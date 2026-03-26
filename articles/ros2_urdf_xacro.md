---
title: "ROS2で自作ロボットをRviz上に3D表示する方法について【URDF/Xacro】"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ros2", "ros", "urdf", "python", "robot"]
published: true
published_at: 2026-03-31 06:00
publication_name: "secondselection"
---

## はじめに

今回は、Rviz（ROS Visualization）上で自作ロボットを3D表示するための環境構築方法についてまとめます。この記事を通じて、URDFやXacroによるロボットモデルの記述からRvizでの可視化までを習得できます。また、ここで構築する環境は地図作成＋自己位置推定（SLAM）や自律走行（Nav2）へ応用できます。

:::message

**Rviz** はROS2に付属する3D可視化ツールです。ロボットのモデルやセンサーデータ、座標系などをリアルタイムで視覚的に確認できます。

:::

この記事では主に以下の内容を扱います。

1. ロボット3D表示用のシンプルなROSパッケージの構築手順
2. STLモデルを使ったURDF/Xacroファイルの記述方法

## 前提知識

- Linuxの基本的なターミナル操作
- ROSの基礎的な知識

ROS2自体の環境構築方法および必要パッケージのインストールについてはこの記事では割愛します。なお、自分の場合はWSL2 + DockerでROS2 Humbleの環境を構築しています。

基本的な用語（ノード、トピック、パッケージ等）については前回の記事を参考にしてください。

@[card](https://zenn.dev/secondselection/articles/ros2_lightrover)

## URDF/Xacro ファイルとは？

ROS2で使われる**URDFやXacroは、ロボットの構造・形状をテキスト（XML形式）で記述**するためのファイルです。

**URDF（Unified Robot Description Format）**
ROS標準のロボット記述フォーマットです。「リンク（部品）」と「ジョイント（関節）」を定義し、部品同士の親子関係やジョイントの種類（固定・可動など）を記述します。

**Xacro（XML Macros）**
URDFを効率的に記述するためのマクロ拡張版です。例えば、ロボットの構造が複雑になると同じような記述を何度もコピー&ペーストすることになります。Xacroでは変数やループ構文を使えるため、記述を大幅に簡略化できます。

## ROSパッケージのディレクトリ構成

今回作成する可視化環境用パッケージのディレクトリ構成は以下のとおりです。

```text
ros2_ws　                        ・・・ ROS 2 のワークスペース
└── src
    └── robot_description        ・・・ ロボットの外観・形状を定義するパッケージ
        ├── launch
        │   └── plant_robot_launch.py    ・・・ Xacroファイル読込、Rviz起動
        ├── meshes                        
        │   └── plant_robot_model.stl    ・・・ ロボットのモデル
        ├── rviz
        │   └── plant_robot.rviz        ・・・ Rvizの表示設定ファイル
        ├── urdf
        │   └── plant_robot.xacro       ・・・ ロボットの構造・関節・位置などを定義
        ├── CMakeLists.txt               ・・・ 依存パッケージ・インストール設定
        └── package.xml                  ・・・ パッケージのメタ情報
```

## 前準備

### ROSパッケージの作成

`ros2_ws/src` ディレクトリに移動後、`ros2 pkg create` でパッケージのテンプレートを作成し、必要なディレクトリを追加します。

```bash:bash
~/ros2_ws/src$ ros2 pkg create --build-type ament_cmake robot_description
~/ros2_ws/src$ cd robot_description
~/ros2_ws/src/robot_description$ mkdir urdf meshes launch rviz
```

### CMakeLists.txtの修正

`CMakeLists.txt` には、プロジェクト名・CMakeバージョン・依存パッケージ・リソースのインストール先などを記述します。`install(DIRECTORY ...)` を記述することで、URDF・launch・meshesなどのファイルがインストール先の `share/robot_description/` に配置されます。他パッケージから `get_package_share_directory('robot_description')` でパスを取得してアクセスできるようになります。

`ros2 pkg create` で生成されたテンプレートの `CMakeLists.txt` にはテスト用の記述などが含まれています。今回はURDFを表示するだけのシンプルな構成なので、テンプレートの内容をすべて削除し、以下の内容に置き換えます。

:::details CMakeLists.txt

```text:text
cmake_minimum_required(VERSION 3.8)
project(robot_description)

find_package(ament_cmake REQUIRED)

install(DIRECTORY urdf launch meshes rviz
  DESTINATION share/${PROJECT_NAME}
)

ament_package()
```

:::

### package.xmlの修正

`package.xml` にはパッケージの情報・依存関係・ライセンスを記述します。実行時に必要なパッケージ（`xacro`、`rviz2` など）もここで宣言します。テンプレートの`package.xml`の内容をすべて削除し、以下の内容に置き換えます。

:::details package.xml

```xml:xml
<package format="3">
  <name>robot_description</name>
  <version>0.0.0</version>
  <description>Robot description package with URDF and meshes</description>
  <maintainer email="your@email.com">Your Name</maintainer>
  <license>Apache-2.0</license>

  <buildtool_depend>ament_cmake</buildtool_depend>

  <exec_depend>xacro</exec_depend>
  <exec_depend>rviz2</exec_depend>

  <export>
    <build_type>ament_cmake</build_type>
  </export>
</package>
```

:::

### launch ファイルの作成

**launchファイルとは、ROS2で複数のノードや設定をまとめて起動するためのファイル**です。  
ROS2ではPythonで記述するのが主流です。

**1コマンドで複数のノードを一括起動できる点が大きなメリット**です。以下のlaunchファイルは、ロボットモデルの可視化環境を立ち上げるために次の2つを同時に起動します。

1. **robot_state_publisher**
   - launchファイルで展開したURDFを受け取り `/robot_description` に公開する
   - `/joint_states` を購読してジョイントの状態を **tf（座標変換情報）** としてブロードキャストする

   :::message
   **tf（Transform）** とは「ある座標系から見た別の座標系の位置と向き」を表す仕組みです。例えば `world → base_link` のtfは、固定された基準座標系（原点）からロボット中心の座標系への変換を表します。これによりRviz上でロボットの位置・姿勢を表現できます。
   :::

2. **rviz2** — 指定した設定ファイルでRvizを起動

:::details plant_robot_launch.py

```python:python
from launch import LaunchDescription
from launch_ros.actions import Node
from ament_index_python.packages import get_package_share_directory
import os
import xacro

def generate_launch_description():
    pkg_share = get_package_share_directory('robot_description')
    xacro_file = os.path.join(pkg_share, 'urdf', 'plant_robot.xacro')

    # xacro を展開して robot_description 文字列を生成
    robot_description_config = xacro.process_file(xacro_file).toxml()

    # robot_state_publisher
    rsp_node = Node(
        package='robot_state_publisher',
        executable='robot_state_publisher',
        name='robot_state_publisher',
        output='screen',
        parameters=[{'robot_description': robot_description_config}]
    )

    # rviz2
    rviz_node = Node(
        package='rviz2',
        executable='rviz2',
        name='rviz2',
        output='screen',
        arguments=['-d', os.path.join(pkg_share, 'rviz', 'plant_robot.rviz')],
        condition=None
    )

    return LaunchDescription([
        rsp_node,
        rviz_node
    ])
```

:::

## 自分で設計したロボットを URDF/Xacro に取り込む

このセクションでは、3D CADで設計したロボットモデルをSTLファイルとして書き出し、URDF/Xacroに取り込んでRvizで表示するまでの手順を説明します。

:::message alert

実際に作業してみて、以下の点で躓きました。

- STLモデルとROS環境での位置とスケールが違うため、そのままでは取り込めなかった
- Xacroファイル内の数値の変更だけでは位置・向きの調整が難しく、意図した配置にならなかった
:::

### STL ファイルの作成

まずFusion360などの3D CADソフトで設計し、作成したモデルを**STLファイル**として書き出します。**STLファイルは3Dモデルの形状を小さな三角形の集合（ポリゴンメッシュ）で表現するファイル形式**です。

![Fusion360でのモデル設計例](/images/ros_urdf/image-1.png)

### パーツ単位で STL ファイルを分ける

Rviz上でパーツを個別に動かしたい場合は、パーツごとにSTLファイルを分ける必要があります。例えば車輪を動かしたいなら、ボディと車輪を別々のSTLファイルとして保存します。また、LiDARやカメラなどのセンサーを搭載した際、センサーを別リンクとして定義し独自のトピックを配信する場合は、センサーごとにSTLファイルを分ける必要があります。

**今回は簡略化のため、ボディのパーツはすべて一体化したモデル**とします。

### 位置座標の調整とスケーリング

ROSにSTLファイルを取り込む前に、**スケールを合わせる必要**があります。

**URDF/Xacroはメートル単位を使うため、ミリメートル単位のCADデータは0.001倍に変換する必要**があります。今回は **Blender** を使って視覚的に位置調整とスケーリングを行いました。Xacroファイル内で数値を手で合わせるより直感的で楽です。

![Blenderでのスケール・位置調整](/images/ros_urdf/image-2.drawio.png)

### URDF/Xacro で STL を指定する

**Blenderで編集したSTLファイル**を `robot_description/meshes` ディレクトリに配置します。`plant_robot.xacro` の `<geometry>` 内に `<mesh>` タグでパスを指定します。

```text
ros2_ws
└── src
    └── robot_description
        ├── launch
        │   └── plant_robot_launch.py
        ├── meshes                        
        │   └── plant_robot_model.stl　　★ STLモデルを配置  
        ├── rviz
        │   └── plant_robot.rviz
        ├── urdf
        │   └── plant_robot.xacro        ★ STLファイルのパスを指定
        ├── CMakeLists.txt
        └── package.xml
```

今回は固定座標系 `world` とロボット本体 `base_link` を定義し、ジョイントで結びます。

Xacroのジョイント定義（`<joint type="...">`）には以下の種類があります。今回はボディを固定するだけの構成なので `fixed` を使用します。

| type | 概要 | 主な用途 |
| ------ | ------ | ---------- |
| `fixed` | 移動・回転なし（剛体結合） | フレームとボディの固定、センサーの取り付け |
| `revolute` | 指定軸まわりの回転（角度制限あり） | アームの関節、ステアリング |
| `continuous` | 指定軸まわりの回転（角度制限なし） | 車輪、プロペラ |
| `prismatic` | 指定軸方向の直線移動（上下限あり） | リニアアクチュエータ、昇降機構 |
| `floating` | 6自由度（位置+姿勢すべて自由） | 空中・水中ロボット |
| `planar` | 指定軸に垂直な平面上の移動 | 平面移動ロボット |

以下の`plant_robot.xacro`では `world_to_base` ジョイントに `type="fixed"` を指定し、`world` と `base_link` を固定しています。

位置とスケールはBlender側で調整済みのため、Xacro内の `origin` はすべてゼロのままにしています。Xacro内で位置を調整する場合は `origin xyz="0 0 0"`（メートル）、向きを調整する場合は `rpy="0 0 0"`（ラジアン）を変更してください。

:::details plant_robot.xacro

```xacro:xacro
<?xml version="1.0"?>
<robot name="plant_robot" xmlns:xacro="http://www.ros.org/wiki/xacro">

  <link name="world"/>
  <joint name="world_to_base" type="fixed">
    <parent link="world"/>
    <child link="base_link"/>
    <origin xyz="0 0 0" rpy="0 0 0"/>
  </joint>

  <!-- ロボット本体 -->
  <link name="base_link">
    <visual>
      <origin xyz="0 0 0" rpy="0 0 0"/>
      <geometry>
        <mesh filename="package://robot_description/meshes/plant_robot_model.stl" scale="1 1 1"/>
      </geometry>
    </visual>
  </link>

</robot>
```

:::

ビルドしてlaunchファイル`plant_robot_launch.py`を起動します。

```bash:bash
~/ros2_ws$ colcon build
~/ros2_ws$ source install/setup.bash
~/ros2_ws$ ros2 launch robot_description plant_robot_launch.py
```

`plant_robot_launch.py`内でRvizを起動するようにしているため、下記の画面が表示されます。  
初回起動時は何もディスプレイに表示されない設定となっています。
![Rviz初期表示](/images/ros_urdf/image-4.png)

ロボットのモデルをRvizのディスプレイへ追加するため、下記の手順を実施します。

1. 「Fixed Frame(原点・固定基準)」を `world` に設定します
![Rviz2の設定1](/images/ros_urdf/image-5.drawio.png)

2. 画面左下の「Add」ボタンから「RobotModel」を選択します
![Rviz2の設定2](/images/ros_urdf/image-6.drawio.png)

3. 「RobotModel」の「Description Topic」に`/robot_description`を購読するように設定します
![Rviz2の設定3](/images/ros_urdf/image-7.drawio.png)

4. Rviz上にロボットのモデルが表示されます
![Rviz2上のロボットモデルの表示](/images/ros_urdf/image-8.drawio.png)

:::message

WSL2では、ROS2の通信不具合により、Rviz上にモデルを表示できないことがありました。
ROS2の通信はDDS（Data Distribution Service）というミドルウェアを使っています。
WSL2は「仮想ネットワーク」上で動いているため、マルチキャスト通信の動作に支障をきたすことがあります。ノード検出にマルチキャストを使用しないCycloneDDSを導入することで、WSL2でも安定して動かすことができましたので試してみてください。

```bash:bash
~/ros2_ws$ sudo apt install ros-$ROS_DISTRO-rmw-cyclonedds-cpp
~/ros2_ws$ echo "export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp" >> ~/.bashrc
~/ros2_ws$ source ~/.bashrc
```

:::

### Rviz 設定ファイルを保存して起動を簡略化する

Rvizの設定（Fixed Frameや表示対象など）は起動のたびに設定がリセットされるため、毎回設定し直す必要があります。これを回避するには、**Rviz上部メニューの [File → Save Config As] で現在の設定を `.rviz` ファイルとして保存し、launchファイル起動時に読み込むように指定**します。

今回のlaunchファイルではすでに `rviz/plant_robot.rviz` を読み込む設定にしているので、そこに保存するだけで次回から設定が維持されます。

## サンプルスクリプトで tf を配信してモデルを動かしてみる

サンプルスクリプトを実行し、キーボード操作で`tf`を配信してRviz上のモデルを動かします。

前述したとおり、**tfは「ある座標系から見た別の座標系の位置と向き」を表します**。今回配信するのは `world → base_link` のtfです。`world` は固定された基準座標系（原点）、`base_link` はロボット中心の座標系なので、`tf`を配信することで原点からのロボットの位置・姿勢を表現できます。

なお、ここで配信している`tf`はあくまで計算値です。実機ではオドメトリのデータを積分して`tf`を生成します。また、SLAMやAMCLなどの自己位置推定を使う場合は、その結果からも`tf`を生成します。

:::message

URDFには `world_to_base` ジョイント（fixed）が定義されています。このため `robot_state_publisher` は `/tf_static` に `world → base_link` の静的tfを配信し続けています。テストスクリプトも同じ `world → base_link` を `/tf` に配信するため、TF2上で二重配信になりますが、動的tf（`/tf`）が常に上書きするため動作上は問題ありません。

:::

:::details test_tf_publisher.py（キーボードでtfを配信するサンプルスクリプト）

```python:python
#!/usr/bin/env python3
import rclpy
from rclpy.node import Node
from geometry_msgs.msg import TransformStamped
import tf2_ros
import threading
import sys
import termios
import tty
import time

class KeyboardAccelNode(Node):
    def __init__(self):
        super().__init__('keyboard_accel_node')

        self.tf_broadcaster = tf2_ros.TransformBroadcaster(self)

        # base_link の位置と速度
        self.x = 0.0
        self.y = 0.0
        self.z = 0.0
        self.vx = 0.0
        self.vy = 0.0

        # 加速度設定
        self.accel = 0.2    # m/s^2
        self.friction = 0.1 # 減速率

        # キー状態
        self.key_state = {'w': False, 's': False, 'a': False, 'd': False}

        # キーボード入力スレッド
        thread = threading.Thread(target=self.keyboard_loop, daemon=True)
        thread.start()

        # TF送信タイマー
        self.dt = 0.05
        self.timer = self.create_timer(self.dt, self.send_tf)  # 20Hz

    def keyboard_loop(self):
        print("Use WASD to move: w=forward, s=back, a=left, d=right, q=quit")
        fd = sys.stdin.fileno()
        old_settings = termios.tcgetattr(fd)
        tty.setcbreak(fd)

        try:
            while True:
                key = sys.stdin.read(1)
                if key in self.key_state:
                    self.key_state[key] = True
                elif key == 'q':
                    print("Exiting")
                    rclpy.shutdown()
                    break
        finally:
            termios.tcsetattr(fd, termios.TCSADRAIN, old_settings)

    def send_tf(self):
        # 加速度計算
        ax = 0.0
        ay = 0.0
        if self.key_state['w']:
            ax += self.accel
        if self.key_state['s']:
            ax -= self.accel
        if self.key_state['a']:
            ay += self.accel
        if self.key_state['d']:
            ay -= self.accel

        # 速度更新
        self.vx += ax * self.dt
        self.vy += ay * self.dt

        # 摩擦で減速
        self.vx *= (1 - self.friction)
        self.vy *= (1 - self.friction)

        # 位置更新
        self.x += self.vx * self.dt
        self.y += self.vy * self.dt

        # TF送信
        tfs = TransformStamped()
        tfs.header.stamp = self.get_clock().now().to_msg()
        tfs.header.frame_id = 'world'    
        tfs.child_frame_id = 'base_link'
        tfs.transform.translation.x = self.x
        tfs.transform.translation.y = self.y
        tfs.transform.translation.z = self.z
        tfs.transform.rotation.x = 0.0
        tfs.transform.rotation.y = 0.0
        tfs.transform.rotation.z = 0.0
        tfs.transform.rotation.w = 1.0
        self.tf_broadcaster.sendTransform(tfs)

        # キー状態リセット
        for k in self.key_state:
            self.key_state[k] = False


def main(args=None):
    rclpy.init(args=args)
    node = KeyboardAccelNode()
    try:
        rclpy.spin(node)
    except KeyboardInterrupt:
        pass
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()

```

:::

スクリプトを実行して`tf`を配信すると、Rviz上でロボットモデルをキーボードで動かせます。

![tf配信で座標変換している様子](/images/ros_urdf/gif-1.gif)

## まとめ

今回はROS2におけるロボットモデルの可視化環境を構築し、自分で設計したモデルをROS環境に取り込むことができました。

今後の課題は以下のとおりです。

- **実機対応** — 実機のセンサーから取得した値を使って動的tfを配信し、Rviz上のモデルと連動させる
- **STL モデルの細分化** — GazeboやGenesisでシミュレーションする場合は車輪の回転などモデル化する必要があるため、パーツをさらに細かく分割する

ロボットの外観・形状をROSへ取り込めるようになれば、SLAMやNav2への応用に近づくため、引き続き実機での実装を進めていきます。

## 参考リンク・文献

@[card](https://wiki.ros.org/ja)
@[card](https://zenn.dev/tasada038/articles/940ef193a4a28a)
[ROS2とPythonで作って学ぶAIロボット入門 (KS理工学専門書)](https://www.amazon.co.jp/ROS2%E3%81%A8Python%E3%81%A7%E4%BD%9C%E3%81%A3%E3%81%A6%E5%AD%A6%E3%81%B6AI%E3%83%AD%E3%83%9C%E3%83%83%E3%83%88%E5%85%A5%E9%96%80-KS%E7%90%86%E5%B7%A5%E5%AD%A6%E5%B0%82%E9%96%80%E6%9B%B8-%E5%87%BA%E6%9D%91-%E5%85%AC%E6%88%90/dp/4065289564?tag=googhydr-22&source=dsa&hvcampaign=books&gad_source=1)
