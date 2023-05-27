## 【OpenCV + Python】LINE Notify で動体検知を知らせる

### 動作環境

- OS: macOS Ventura Version13.3.1
- Python 3.9.13
- OpenCV 4.7.0

OpenCV は pip install opencv-python でインストールできます。

### ソースコード

```python
import cv2
import time
import requests
import json

# LINE Notify token (replace with your own)
LINE_NOTIFY_TOKEN = "YOUR_LINE_NOTIFY_TOKEN"

# movie = cv2.VideoCapture('./movie/park.mp4')
movie = cv2.VideoCapture(0)
red = (0, 0, 255) # 枠線の色
before = None # 前回の画像を保存する変数
fps = int(movie.get(cv2.CAP_PROP_FPS)) #動画のFPSを取得

while True:
    # 画像を取得
    ret, frame = movie.read()
    # 再生が終了したらループを抜ける
    if ret == False: break
    # 白黒画像に変換
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)

    if before is None:
        before = gray.astype("float")
        continue
    #現在のフレームと移動平均との差を計算
    cv2.accumulateWeighted(gray, before, 0.8)
    frameDelta = cv2.absdiff(gray, cv2.convertScaleAbs(before))
    #frameDeltaの画像を２値化
    thresh = cv2.threshold(frameDelta, 3, 255, cv2.THRESH_BINARY)[1]
    #輪郭のデータを得る
    contours = cv2.findContours(thresh,
                    cv2.RETR_EXTERNAL,
                    cv2.CHAIN_APPROX_SIMPLE)[0]

    # 差分があった点を画面に描く
    for target in contours:
        x, y, w, h = cv2.boundingRect(target)
        if w < 30: continue # 小さな変更点は無視
        cv2.rectangle(frame, (x, y), (x+w, y+h), red, 2)

        # Send LINE notification when motion is detected
        payload = {"message": "Motion detected!"}
        headers = {"Authorization": "Bearer " + LINE_NOTIFY_TOKEN}
        requests.post("https://notify-api.line.me/api/notify", data=payload, headers=headers)

    #ウィンドウでの再生速度を元動画と合わせる
    time.sleep(1/fps)
    # ウィンドウで表示
    cv2.imshow('target_frame', frame)
    # Enterキーが押されたらループを抜ける
    if cv2.waitKey(1) == 13: break

#内蔵カメラを解放する
movie.release()
cv2.destroyAllWindows() # ウィンドウを破棄

```

```
movie = cv2.VideoCapture(0)
```

引数で使用カメラを変更できます。(0 はデフォルトで内蔵カメラ)

```
YOUR_LINE_NOTIFY_TOKEN
```

に LINE Notify のアクセストークンを代入します。

LINE Notify アクセストークンの発行方法はこちら

https://firestorage.jp/business/line-notify/

### 結果

![](https://storage.googleapis.com/zenn-user-upload/1a8f0a3ec461-20230524.png)

動体検知したらこんな感じに枠線でトラッキングし、LINE Notify で知らせてくれた。

### 感想

過剰に検出して通知してしまうのを程よく抑えるのが苦労した。

学習率を変更したり、フレーム間の差分の閾値を変えたりして、試行錯誤した。
自分の環境では、学習率を 0.975 くらいに設定してちょうどよく動体検知してくれた。

録画されない ZOOM の授業とかを動体検知して、スライドが動いたらスクショするプログラムとかも作れるかもしれない。
