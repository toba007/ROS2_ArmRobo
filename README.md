[README.md](https://github.com/user-attachments/files/27154844/README.md)
# myCobot 280 × ROS2 Jazzy セットアップガイド (WSL2)

Windows (WSL2) 上で myCobot 280 M5 を ROS2 Jazzy で動かすまでの手順をまとめたリポジトリです。

## 動作環境

| 項目 | バージョン |
|---|---|
| OS | Windows 11 + WSL2 |
| Ubuntu | 24.04 LTS (Noble) |
| ROS | ROS2 Jazzy |
| ロボット | myCobot 280 M5 |
| Python | 3.12 |

---

## 1. ROS2 Jazzy のインストール

### ロケール設定

```bash
sudo apt update && sudo apt install locales
sudo locale-gen en_US en_US.UTF-8
sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
export LANG=en_US.UTF-8
```

### リポジトリ追加

```bash
sudo apt install software-properties-common curl
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] \
  http://packages.ros.org/ros2/ubuntu noble main" | \
  sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null
```

### インストール

```bash
sudo apt update
sudo apt install ros-jazzy-desktop
```

### 環境設定

```bash
echo "source /opt/ros/jazzy/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

> ⚠️ すでに `humble` の設定が `.bashrc` にある場合は先に削除する
> ```bash
> sed -i '/ros\/humble/d' ~/.bashrc
> ```

---

## 2. pymycobot のインストール

Ubuntu 24.04 は PEP 668 によりシステムへの `pip` インストールが制限されています。

```bash
pip3 install pymycobot --break-system-packages
pip3 install pyserial --break-system-packages
```

---

## 3. mycobot_ros2 ワークスペースのセットアップ

```bash
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws/src
git clone https://github.com/elephantrobotics/mycobot_ros2.git
```

### 依存パッケージのインストール・ビルド

```bash
cd ~/ros2_ws
sudo apt install python3-colcon-common-extensions python3-rosdep -y
sudo rosdep init
rosdep update
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
```

### ワークスペースを有効化

```bash
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc
```

---

## 4. myCobot 本体のファームウェア書き込み

ROS から実機を操作するには、本体に以下のファームウェアを書き込む必要があります。

1. [myStudio](https://www.elephantrobotics.com/en/downloads/) を Windows にインストール
2. myCobot を USB で Windows に接続
3. myStudio を起動し、以下を書き込む

| 部位 | ファームウェア |
|---|---|
| M5Stack 本体 | `minirobot` → `Transponder` |
| Atom（先端） | `AtomMain` |

---

## 5. WSL2 への USB シリアル接続（usbipd）

WSL2 は USB デバイスを直接認識しないため、`usbipd` を使って接続します。

### Windows 側：usbipd のインストール

PowerShell（管理者）で実行：

```powershell
winget install usbipd
```

### デバイスの確認とアタッチ

myCobot を USB 接続した状態で PowerShell（管理者）を開く：

```powershell
# 接続デバイス一覧を確認（CP210x が myCobot）
usbipd list

# BUSID を確認して bind & attach（例: 1-2）
usbipd bind --busid 1-2
usbipd attach --wsl --busid 1-2
```

### WSL 側で確認・権限付与

```bash
ls /dev/ttyUSB*          # /dev/ttyUSB0 が表示されれば OK
sudo chmod 666 /dev/ttyUSB0
```

> ⚠️ WSL を再起動するたびに `usbipd attach` のやり直しが必要です

---

## 6. 起動

### スライダーで実機を制御

```bash
sudo chmod 666 /dev/ttyUSB0
ros2 launch mycobot_280 slider_control.launch.py
```

Rviz とスライダー GUI が起動し、スライダーを動かすと実機が連動します。

### 利用可能な launch ファイル一覧

| ファイル名 | 内容 |
|---|---|
| `slider_control.launch.py` | スライダーで関節制御 |
| `simple_gui.launch.py` | シンプル GUI（実機接続必要） |
| `teleop_keyboard.launch.py` | キーボード操作 |
| `mycobot_follow.launch.py` | フォロー制御 |

---

## トラブルシューティング

| 症状 | 原因 | 対処 |
|---|---|---|
| `/dev/ttyUSB0` が見つからない | usbipd が切断済み | `usbipd attach --wsl --busid X-X` を再実行 |
| `SerialException` | ポート権限不足 | `sudo chmod 666 /dev/ttyUSB0` |
| `externally-managed-environment` | PEP 668 制限 | `--break-system-packages` を付けてインストール |
| `catkin_make` が見つからない | ROS1 コマンド | ROS2 では `colcon build` を使う |
| ロボットが動かない | ファームウェア未書き込み | myStudio で Transponder / AtomMain を書き込む |
| WSL 再起動後に USB 認識しない | usbipd の仕様 | 毎回 `usbipd attach` が必要 |

---

## 参考リンク

- [Elephant Robotics 公式ドキュメント](https://docs.elephantrobotics.com/)
- [mycobot_ros2 リポジトリ](https://github.com/elephantrobotics/mycobot_ros2)
- [ROS2 Jazzy インストール手順](https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html)
- [usbipd-win](https://github.com/dorssel/usbipd-win)
