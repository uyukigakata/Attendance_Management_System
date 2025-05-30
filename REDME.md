## ラズパイ接続しNFCカードリーダーの中身を見る

```jsx
#raspiが繋がっているものの確認識別番号表示
lsusb
...
Bus 001 Device 004: ID 054c:06c1 Sony Corp. RC-S380/S
...
```

## desktop PCで、SONYのカードリーダーの初期設定

```jsx
#library
sudo apt update
sudo apt install libusb-dev 
sudo apt install libpcsclite-dev 
sudo apt install pcscd 
sudo apt install pcsc-tools

pip install nfcpy

```

## nfcpyの初期設定

```jsx
python3 -m nfc
> This is the 1.0.4 version of nfcpy run in Python 3.9.7
on Linux-5.13.0-37-generic-x86_64-with-glibc2.34
I'm now searching your system for contactless devices
** found usb:054c:06c1 at usb:001:017 but access is denied
-- the device is owned by 'root' but you are 'kazukichi'
-- also members of the 'root' group would be permitted
-- you could use 'sudo' but this is not recommended
-- better assign the device to the 'plugdev' group
   sudo sh -c 'echo SUBSYSTEM==\"usb\", ACTION==\"add\", ATTRS{idVendor}==\"054c\", ATTRS{idProduct}==\"06c1\", GROUP=\"plugdev\" >> /etc/udev/rules.d/nfcdev.rules'
   sudo udevadm control -R # then re-attach device
I'm not trying serial devices because you haven't told me
-- add the option '--search-tty' to have me looking
-- but beware that this may break other serial devs
Sorry, but I couldn't find any contactless device
```

と表示される↓

```jsx
#以下を実行
sudo sh -c 'echo SUBSYSTEM==\"usb\", ACTION==\"add\", ATTRS{idVendor}==\"054c\", ATTRS{idProduct}==\"06c1\", GROUP=\"plugdev\" >> /etc/udev/rules.d/nfcdev.rules'
sudo udevadm control -R # then re-attach device
```

## NFCカードリーダーが上手く認識	されているかを確認

```jsx
#コマンドライン時(True表示でOK
python
> import nfc
> clf = nfc.ContactlessFrontend()
> clf.open('usb:054c:06c1')
True
```

## 最終的なコード
main.py

```jsx
import openpyxl as op
import sqlite3
import nfc
import re
import sys
import time
import slackweb

def db_setting(active_sheet, total_row):
    # DBの読み込み
    # 自動コミットモードを設定
    conn = sqlite3.connect("Member.db", isolation_level=None)

    # テーブルを削除するSQL文
    delete_table_sql = """
    DROP TABLE SecMember;
    """
    # テーブルが無い時のエラー処理
    try:
        # SQL文を実行
        conn.execute(delete_table_sql)
    except:
        pass

    # テーブルを作成するSQL文
    create_table_sql = """
    CREATE TABLE SecMember (
        student_id INTEGER(7),
        name TEXT,
        status TEXT
    );
    """
    # SQL文を実行
    conn.execute(create_table_sql)

    for i in range(2, total_row+1):
        student_id = active_sheet.cell(column=1, row=i).value
        name = active_sheet.cell(column=3, row=i).value
        data = (student_id, name, "exit")
        sql = "INSERT INTO SecMember (student_id, name, status) VALUES (?, ?, ?)"
        conn.execute(sql, data)
        
    conn.close()

def show_table():
    # DBの読み込み
    # 自動コミットモードを設定
    conn = sqlite3.connect("Member.db", isolation_level=None)
    c = conn.cursor()
    c.execute("SELECT * FROM SecMember")

    print("-----SecMember Table-----")

    for row in c:
        print(row)
    
    conn.close()
    
def nfc_loading():
    conn = sqlite3.connect("Member.db")     # データベース接続
    slack = slackweb.Slack(url="https://hooks.slack.com/services/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")
    clf = nfc.ContactlessFrontend("usb")    # USB接続のNFCリーダーを開く
    prev_time = 0   # 前の学生証タッチ時間
    prev_info = None    # 前の学生証の情報

    # finallyを記述
    try:
        while(True):
            # NFCタグのタッチ認識を開始
            tag = clf.connect(rdwr={'on-connect': lambda tag: False})
            
            # タッチが弱くて読み取れない時のエラー検出
            try:
                nfc_info = tag.dump()   # NFCタグの中身を取り出す
            except nfc.tag.tt3.Type3TagCommandError:
                continue

            the_time = time.time()  # 現在のUnix時間を取得
            dif_time = the_time - prev_time     # 現在と前回のタッチ時間の差を取得
            prev_time = the_time    # 現在のUnix時間を退避させておく

            # カードの情報が前回と同じ ＆ 前回とのタッチ時間の差が3秒より小さい場合, 最初に戻る
            if(nfc_info == prev_info and dif_time < 3):
                prev_info = nfc_info    # 現在のカード情報を退避させておく
                continue

            prev_info = nfc_info    # 現在のカード情報を退避させておく

            # カードが学生証ではなかった場合のエラーを検出
            try:
                _str = nfc_info[4]
                start = _str.index('|') + 2
                end = start + 7
                student_id = [int(_str[start:end])]     # 学籍番号を取得
            except Exception as e:
                continue

            # データベースに学籍番号があるか検索するSQL文
            search_sql = "SELECT name, status FROM SecMember WHERE student_id=?"
            
            # SQL文を実行
            db_info = conn.execute(search_sql, student_id)
            
            # データベースから取得したデータを分解
            for row in db_info:
                name = row[0]   # 氏名を取得
                status = row[1]     # 入退室状況を取得

            try:
                # 入退室状況に応じてステータスを変更する
                if(status == "exit"):
                    slack.notify(text=name + "さんが入室しました")  # Slackに通知
                    update_sql = "UPDATE SecMember SET status='enter' WHERE student_id=?"   # 入退室状況を変更するSQL文
                else:
                    slack.notify(text=name + "さんが退室しました")  # Slackに通知
                    update_sql = "UPDATE SecMember SET status='exit' WHERE student_id=?"    # 入退室状況を変更するSQL文
            except Exception:
                continue

            conn.execute(update_sql, student_id)    # SQL文を実行
            conn.commit()   # SQLの処理を確定

    # 最後に必ず実行される処理
    finally:
        conn.close()    # データベースへの接続を切断
    
def main():
    # Excelデータの読み込み
    wb = op.load_workbook("members.xlsx")
    # シートをアクティブ状態にする
    active_sheet = wb.active
    # Excelの行数を取得
    total_row = active_sheet.max_row
    
    db_setting(active_sheet, total_row)
    show_table()
    
    nfc_loading()

    
if __name__ == "__main__":
    main()

```

## Member date

members.py(Excelシート作成)

```jsx
import openpyxl

# 新しいExcelファイルを作成
wb = openpyxl.Workbook()
sheet = wb.active

# ヘッダー行を作成
sheet['A1'] = '学籍番号'
sheet['B1'] = '無視する列'
sheet['C1'] = '氏名'

# データ行を追加
data = [
    (1204269, '無視', '高垣 優'),
    (2023002, '無視', '鈴木 花子'),
    (2023003, '無視', '佐藤 次郎'),
    (2023004, '無視', '高橋 健')
]

for idx, (student_id, dummy, name) in enumerate(data, start=2):
    sheet.cell(row=idx, column=1, value=student_id)
    sheet.cell(row=idx, column=2, value=dummy)
    sheet.cell(row=idx, column=3, value=name)

# 保存
wb.save('members.xlsx')
print("members.xlsxを作成しました！")

```

## NFCカードリーダーがICを読み込むのかを判断

### sample code

- NFCリーダーでICカードのチップを読み取れるかを判断

```jsx
import nfc

def on_connect(tag):
    print("Tag detected!")
    print(tag)
    return False  # 1回検出したらすぐ終了

clf = nfc.ContactlessFrontend('usb')
clf.connect(rdwr={'on-connect': on_connect})
clf.close()

```

## 常時コードを実行する設定

### サービスファイル作成

```jsx
sudo nano /etc/systemd/system/nfc-reader.service
```

```jsx
[Unit]
#serviceの起動タイミング・依存関係
Description=NFC Reader Service #サービスの説明名(なんでも
After=network.target pcscd.socket #ネットワーク起動かつPC/SCデバイス接続時に
Requires=pcscd.socket #pcscd必須

[Service]
#実際に何を実行するか環境ルール設定
ExecStart=/home/kitsec/card/venv/bin/python3 /home/kitsec/card/main.py #環境変数＆codeパス設定
WorkingDirectory=/home/kitsec/card/ #実行する作業ディレクトリパス
Restart=always #プログラムが落ちても自動的に再起動
User=kitsec #Linuxユーザー
Group=kitsec

[Install]
#サービス起動時に紐づける場所
WantedBy=multi-user.target #Rasberry Pi起動時に実行

```

## serviceが止まった時確認すること

### 1. serviceが停止していないかを確認

```jsx
sudo systemctl status "サービス名"

sudo systemctl status nfc-reader.service
sudo systemctl status pcscd

```

### 2. serviceを永続化

```jsx
sudo systemctl enable "サービス名"

sudo systemctl enable nfc-reader.service
sudo systemctl enable pcscd

```