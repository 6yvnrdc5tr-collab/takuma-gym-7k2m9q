#!/usr/bin/env python3
"""
札幌市公共施設予約 空き監視 → スマホ通知（通知専用）

- 条件: 利用目的=バレーボール / 全9区 / 夜間 / 日付は引数で指定
- 空きが出た体育館を ntfy でスマホに即通知（タップで検索結果ページが開く）
- ログイン・仮予約・確定は自分で手動。自動で予約操作はしません。

準備:
  pip install playwright
  playwright install chromium
  スマホに ntfy アプリを入れ、下の NTFY_TOPIC と同じ名前を購読

使い方:
  python gym_watch.py --test-notify
  python gym_watch.py --dates 2026-10-20 --once --debug      # 初回確認用
  python gym_watch.py --dates 2026-10-20,2026-10-27..2026-10-31
"""
import argparse
import datetime as dt
import json
import os
import random
import sys
import time
import urllib.request
from urllib.parse import urlencode

from playwright.sync_api import sync_playwright

# ===== 設定 =====
NTFY_SERVER = "https://ntfy.sh"
NTFY_TOPIC = os.environ.get("NTFY_TOPIC", "takuma-gym-CHANGE-ME-8f3k2")  # 環境変数があればそちらを優先
INTERVAL_SEC = 7        # 1リクエストごとの間隔（秒）
PURPOSE = "70"          # 利用目的（URLのu[0]）= バレーボール
TIME_SLOT = "2"         # 時間帯（URLのpt[0]）= 夜間
WARD_CODES = ["10", "20", "30", "40", "45", "50", "55", "60", "70"]  # 全9区
BASE = "https://yoyaku.harp.lg.jp/sapporo/FacilitySearch/Index"
# ================

JS_EXTRACT = """
() => {
  const skip = ['凡例', '該当する施設', '選択済み', '{{'];
  const out = {available: [], unavailable: [], marker: false};
  let inError = false;
  for (const h of document.querySelectorAll('h3, h4')) {
    const t = h.innerText.trim().replace(/\\s+/g, ' ');
    if (!t) continue;
    if (t.includes('希望日時に予約できません')) { inError = true; out.marker = true; continue; }
    if (skip.some(s => t.includes(s))) continue;
    (inError ? out.unavailable : out.available).push(t);
  }
  out.text = document.body.innerText;
  return out;
}
"""


def build_url(date: str) -> str:
    params = [("u[0]", PURPOSE)]
    params += [(f"a[{i}]", c) for i, c in enumerate(WARD_CODES)]
    params += [("pt[0]", TIME_SLOT), ("ud", date)]
    return BASE + "?" + urlencode(params)


def parse_dates(arg: str):
    today = dt.date.today()
    dates = []
    for part in arg.split(","):
        part = part.strip()
        if not part:
            continue
        if ".." in part:
            a, b = part.split("..")
            d, end = dt.date.fromisoformat(a), dt.date.fromisoformat(b)
            while d <= end:
                dates.append(d)
                d += dt.timedelta(days=1)
        else:
            dates.append(dt.date.fromisoformat(part))
    return [d.isoformat() for d in sorted(set(dates)) if d >= today]


def notify(title: str, message: str, click: str = None, priority: int = 4):
    body = {"topic": NTFY_TOPIC, "title": title, "message": message, "priority": priority}
    if click:
        body["click"] = click
    req = urllib.request.Request(
        NTFY_SERVER,
        data=json.dumps(body).encode("utf-8"),
        headers={"Content-Type": "application/json"},
    )
    urllib.request.urlopen(req, timeout=15).read()


def fetch(page, date: str, debug: bool):
    resp = page.goto(build_url(date), wait_until="networkidle", timeout=30000)
    status = resp.status if resp else 0
    if status in (403, 429, 503):
        raise RuntimeError(f"HTTP {status}")
    # テンプレート({{ }})が消えて結果が描画されるまで待つ
    page.wait_for_function(
        "() => !document.body.innerText.includes('{{') && /件/.test(document.body.innerText)",
        timeout=20000,
    )
    page.wait_for_timeout(500)
    data = page.evaluate(JS_EXTRACT)
    if debug:
        print("---- 画面テキスト(先頭1500字) ----")
        print(data["text"][:1500])
        print("---- 空き候補:", data["available"])
        print("---- 不可    :", data["unavailable"])
    return data


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--dates", help="例: 2026-10-20,2026-10-27..2026-10-31")
    ap.add_argument("--once", action="store_true", help="1周だけ実行")
    ap.add_argument("--debug", action="store_true")
    ap.add_argument("--test-notify", action="store_true")
    ap.add_argument("--duration", type=int, default=0, help="この秒数だけ監視して終了（クラウド実行用）")
    ap.add_argument("--state", help="通知済み状態を保存するJSONファイル")
    args = ap.parse_args()

    if args.test_notify:
        notify("テスト通知", "gym_watch.py からの通知テストです")
        print("送信しました")
        return
    if not args.dates:
        ap.error("--dates が必要です")
    dates = parse_dates(args.dates)
    if not dates:
        sys.exit("有効な日付がありません（過去日は除外されます）")

    print(f"監視開始: {dates} / 間隔 {INTERVAL_SEC}s")
    seen = {d: set() for d in dates}  # 通知済み（現在空きのもの）
    if args.state and os.path.exists(args.state):
        try:
            with open(args.state, encoding="utf-8") as f:
                saved = json.load(f)
            for d in dates:
                seen[d] = set(saved.get(d, []))
        except Exception:  # noqa: BLE001
            pass
    deadline = time.time() + args.duration if args.duration else None
    backoff = 0

    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True)
        ctx = browser.new_context(locale="ja-JP", timezone_id="Asia/Tokyo")
        page = ctx.new_page()
        try:
            while True:
                for d in dates:
                    if deadline and time.time() >= deadline:
                        break
                    try:
                        data = fetch(page, d, args.debug)
                        backoff = 0
                    except Exception as e:  # noqa: BLE001
                        backoff = min(max(backoff * 2, 30), 600)
                        print(f"[{d}] 取得失敗: {e} → {backoff}s 待機")
                        time.sleep(backoff)
                        continue

                    now = set(data["available"])
                    new = now - seen[d]
                    seen[d] = now  # 空きが消えたら次回また通知対象になる
                    stamp = dt.datetime.now().strftime("%H:%M:%S")
                    print(f"[{stamp}] {d} 空き{len(now)}件 / 新規{len(new)}件")
                    if new:
                        names = "\n".join(sorted(new))
                        try:
                            notify(f"空き: {d} 夜間", names, click=build_url(d))
                        except Exception as e:  # noqa: BLE001
                            print("通知失敗:", e)
                    time.sleep(INTERVAL_SEC + random.uniform(0, 1))
                if args.once or (deadline and time.time() >= deadline):
                    break
        except KeyboardInterrupt:
            print("停止しました")
        finally:
            if args.state:
                with open(args.state, "w", encoding="utf-8") as f:
                    json.dump({d: sorted(v) for d, v in seen.items()}, f, ensure_ascii=False)
            browser.close()


if __name__ == "__main__":
    main()
