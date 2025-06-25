---
title: "ROS2でライトローバーを自律走行させてみた"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ros2", "ros", "nav2", "python"]
published: true
published_at: 2025-06-30 09:00
publication_name: "secondselection"
---

## はじめに

- この記事は「ROS（Robot Operating System）」に興味がある方、名前は知っているけど詳しくは分からない初心者の方向けです。  

- 必要な前提知識として、Ubuntuのインストール方法や基本的ターミナル操作方法、マイクロコンピュータ（Raspberry Pi）の知識が必要です。

- この記事では、ヴィストン株式会社が販売しているライトローバーを使用し、自律走行させるまでに必要なROSの環境構築の方法や知識をまとめてます。

## 自律走行するロボットとは？

自律走行するロボットとは、外部からの指示や人間の操作なしに、自らのセンサ情報を基に周囲の状況を認識して移動するロボットのことです。
自律走行には「周辺認識」「判断」「制御」の3つの主要なステップが含まれます。自律走行を実現するためには、これらのステップを支えるセンサや機能がロボットに搭載されている必要があります。

1. **周辺認識（自己位置推定＆地図生成）**
    1. LiDAR、エンコーダ、カメラ、IMUなどのセンサ
    2. SLAMと呼ばれる自己位置推定と地図作成を同時行う技術

2. **判断（ゴールまでの経路計画、障害物の回避、加減速判断など）**
    1. LiDAR、エンコーダ、カメラ、IMUなどのセンサ
    2. Navigation2と呼ばれるロボットの行動に関する細かなナビゲーションを行えるシステム

3. **制御（直進、後退、停止、旋回、加速、減速など）**
    1. モータなどのアクチュエータ
    2. アクチュエータを制御するモータドライバ、マイクロコンピュータ

## ROS（Robot Operating System）

ROS（Robot Operating System）は、ロボットアプリケーションの開発を支援するためのオープンソースソフトウェアフレームワークです。OSという名称が付いていますが、実際にはロボットのソフトウェア開発に必要なライブラリ、ツール、通信機能などを提供するミドルウェアの役割を果たします。

### ROSを利用するメリット

- オープンソースが豊富なため、ゼロから開発する手間と費用を大幅に削減できます。
- 多様なセンサー(LiDAR、カメラ、IMUなど)と互換性があります。
- ロボットの状態をビューアツールを利用し、リアルタイムに可視化できます。

### ROSのバージョン

ROS（Robot Operating System）は、2025年6月時点では、ROS1とROS2で2つのバージョンが存在しています。ROS1については2025年5月でサポートが終了しており、ROS1からROS2へバージョン移行が進んでいる状況です。

| ROSバージョン | ディストリビューション  | 対応OS |
| ---- | ---- | ---- |
| ROS1 | Noetic, Melodic等 | Ubuntu 20.04まで|
| ROS2 | Humble, Foxy等 |Ubuntu 20.04, 22.04, 24.04, Windows, macOS|

:::message

ROSのディストリビューションとは、特定のROSバージョンに対応したソフトウェアパッケージのまとまりのことです。
ディストリビューション名は、基本的に亀に関係した名前となっています。亀の上に乗った象が大地を支えているとする古代の宇宙観に沿って、知能ロボットの世界においてROSが亀のような役割を果たすようにとの願いが込められているらしいです。

:::

### ROS関連の用語

下記に示す用語やツールを最低限覚えておくと理解が深まると思ったので紹介します。

#### ROSの基本的な用語

- ノード（Node）: ROSにおける基本的な実行単位。個々のプロセスやプログラム。  
例えば, ノードはセンサの読取やモータの制御などのタスクを実行します。

- トピック(Topic)：ノード間でメッセージの送受信に使う名前付きの通信チャネル。  
例えば, /cmd_velや/odomなどの名前が付けられます。

- メッセージ（message）: トピックでやり取りされる構造化されたデータ。
例えば, geometry_msgs/msg/Twistなどがあります。

- パブリッシャー(Publisher)：トピックにメッセージを送信するノード。

- サブスクライバー(Subscriber)：トピックからメッセージを受信するノード。

- サービス（Service）: リクエストとレスポンス形式の同期通信。必要なときに処理を要求します。

- ローンチファイル（launch file）：複数のノードを一括で起動・管理するための設定ファイル

![alt text](/images/node.drawio.png)

#### ROSで使用される技術・ツール

- SLAM（Simultaneous Localization and Mapping）: 自己位置推定と地図作成を同時行う技術。

- TF（Transform）：複数の座標系間の関係（位置・姿勢）を管理するツール、ロボットの各部位の位置などを扱うことができます。

- RViz：リアルタイムでロボットの状態やセンサーデータを可視化するビューアツール。

- Gazebo：ロボットの物理シミュレーションするためのツール。

- Navigation2：ロボットの行動に関する細かなナビゲーションを行えるシステムツール。

## システム構築

### 準備

ROS2でライトローバーの自律走行を行うにあたり、必要なものとそれぞれの役割は下記のとおりです。

1. **ノートPC**　：
    - OS: Ubuntu　22.04 LTS
    - ROSのバージョン: ROS2 Humble

2. **ライトローバー本体**：
![alt text](/images/light_rover.drawio.png)

    - **ソフトウェア**
        - OS: Ubuntu MATE 22.04
        - ROSのバージョン: ROS2 Humble

    - **ハードウェア**
        - マイクロコンピュータ：Raspberry Pi 4B × 1
        - モータ制御基板：VS-WRC201 × 1
        - LiDAR：YDLiDAR X2 × 1
        - モータ：DCモータ（エンコーダ付き）× 2
        - モバイルバッテリー × 1

    **補足**

    - ライトローバーをROS2で動かすと消費電力が大きいため、別途モバイルバッテリーを準備Sします。
    - 開発段階では、電源ケーブル(5V,3A)を使用することを推奨します。
    - 電源はRaspberry PiのUSB-typeCの給電口に接続すると、モータ制御基板経由でモータにも給電されます。
    - LiDARはRaspberry PiのUSB-typeAに接続すると、給電されます。

:::message

LiDARに使用するケーブルは通信用のUSBケーブルを使用してください。
通信用のケーブルだと思って電源用USBケーブルを使った結果、LiDARから値が取得できず、初期設定が悪いと思い込み、泥沼に陥ってしまいました💦。

:::

**各機器の役割**
ライトローバー、ノートPCそれぞれの役割については以下のとおりです。

![alt text](/images/system.drawio.png)

### Raspberry Piの環境構築

ここからはRaspberry PiにUbuntuがインストール済みで、SSH接続が可能であることを前提とします。セットアップはVsCodeの拡張機能「Remote-SSH」でRaspberry Piに接続して実施しています。Raspberry Piだけでセットアップは可能ですが、動作が重いため、SSH接続推奨です。

1. **Raspberry PiにROS2をインストール**
下記に示すコマンドを入力し、Raspberry PiにROS2をインストールします。

    :::details ROS2-Humbleのインストール

    ```bash:bash
    #ROS2インストール
    #ロケールの設定
    sudo apt update && sudo apt install locales
    sudo locale-gen en_US en_US.UTF-8
    sudo update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
    export LANG=en_US.UTF-8

    #ROS2のリポジトリをソースリストに追加
    sudo apt install software-properties-common
    sudo add-apt-repository universe
    sudo apt update && sudo apt install curl -y

    sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg

    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

    #ROS2 Humbleのインストール
    sudo apt update
    sudo apt upgrade
    sudo apt install ros-humble-desktop
    sudo apt install ros-dev-tools

    #環境変数の設定　
    #ターミナル起動する際にROSへのパスが通るように設定する。
    source /opt/ros/humble/setup.bash
    echo "source /opt/ros/humble/setup.bash" >> ~/.bashrc
    ```

    :::

2. **Raspberry Pi上でワークスペースを作成～パッケージのクローン**  
    ROS2用のワークスペースを作成し、ライトローバーのパッケージをクローンします。

    :::details ROS2ワークスペースの作成

    ```bash:bash
    #ディレクトリの作成
    mkdir -p ~/ros2_ws/src

    #lightrover_ros2のパッケージをクローン
    cd ~/ros2_ws/src
    git clone --recursive -b humble https://github.com/vstoneofficial/lightrover_ros2.git
    sudo chmod +x ~/ros2_ws/src/lightrover_ros2/lightrover_ros/lightrover_ros/*.py 
    ```

    :::

3. **パッケージのビルド**

    ros2_wsに移動後、colcon buildと入力し、パッケージのビルドを行います。  

    ```bash:bash
    cd ~/ros2_ws
    colcon build --symlink-install  --cmake-clean-cache  --parallel-workers 2
    ```

4. **Raspberry Pi上でYDLiDAR X2（LiDAR）のセットアップ**  

    次にLiDARのセットアップを行います。 事前にLiDARをRaspberry PiにUSBケーブルで接続しておきます。

    :::details YDLiDAR X2（LiDAR）のセットアップ

    ```bash:bash
    # ホームフォルダへ移動
    cd ~

    # 必要なパッケージのインストール
    sudo apt install cmake pkg-config
    sudo apt install swig python3-pip
    git clone https://github.com/YDLIDAR/YDLidar-SDK.git

    # YDLiDAR SDKのインストール
    cd YDLidar-SDK/
    mkdir build
    cd build/
    cmake ..
    make
    sudo make install

    # YDLiDAR用のROSデバイスドライバパッケージの入手
    cd ~/ros2_ws/src
    git clone https://github.com/YDLIDAR/ydlidar_ros2_driver.git
    ```

    :::

    :::message

    YDLiDAR X2（LiDAR）とはレーザー光を照射し、その反射光の情報から対象物との距離や形状を測定する技術を用いたレーザーセンサです。

    :::

5. **ydlidar_rosパッケージ内のソースコードを修正する。**

    ydlidar_rosパッケージ内にある`/src/ydlidar_ros2_driver_node.cpp`の44行目~156行目あたりを修正します。 `declare_parameter`では、ノードが受け入れるパラメータを宣言しており、各デフォルト値を指定し、初期化しています。ydlidar_rosパッケージ内のソースコードは`declare_parameter`の引数が足りていないため、修正します。

    :::details ydlidar_ros2_driver_node.cppの修正

    ```cpp:ydlidar_ros2_driver_node.cpp 44行~156行

    node->declare_parameter<std::string>("port", "/dev/ttyUSB0");
    node->get_parameter("port", str_optvalue);
    ///lidar port
    laser.setlidaropt(LidarPropSerialPort, str_optvalue.c_str(), str_optvalue.size());

    ///ignore array
    str_optvalue = "";
    node->declare_parameter<std::string>("ignore_array", "");
    node->get_parameter("ignore_array", str_optvalue);
    laser.setlidaropt(LidarPropIgnoreArray, str_optvalue.c_str(), str_optvalue.size());

    std::string frame_id = "laser_frame";
    node->declare_parameter<std::string>("frame_id", "laser_frame");
    node->get_parameter("frame_id", frame_id);

    //////////////////////int property/////////////////
    /// lidar baudrate
    int optval = 230400;
    node->declare_parameter<int>("baudrate", 115200);
    node->get_parameter("baudrate", optval);
    laser.setlidaropt(LidarPropSerialBaudrate, &optval, sizeof(int));
    /// tof lidar
    optval = TYPE_TRIANGLE;
    node->declare_parameter<int>("lidar_type", 1);
    node->get_parameter("lidar_type", optval);
    laser.setlidaropt(LidarPropLidarType, &optval, sizeof(int));
    /// device type
    optval = YDLIDAR_TYPE_SERIAL;
    node->declare_parameter<int>("device_type", 0);
    node->get_parameter("device_type", optval);
    laser.setlidaropt(LidarPropDeviceType, &optval, sizeof(int));
    /// sample rate
    optval = 9;
    node->declare_parameter<int>("sample_rate", 3);
    node->get_parameter("sample_rate", optval);
    laser.setlidaropt(LidarPropSampleRate, &optval, sizeof(int));
    /// abnormal count
    optval = 4;
    node->declare_parameter<int>("abnormal_check_count", 4);
    node->get_parameter("abnormal_check_count", optval);
    laser.setlidaropt(LidarPropAbnormalCheckCount, &optval, sizeof(int));

    /// Intenstiy bit count
    optval = 8;
    node->declare_parameter<int>("intensity_bit", 0);
    node->get_parameter("intensity_bit", optval);
    laser.setlidaropt(LidarPropIntenstiyBit, &optval, sizeof(int));

    //////////////////////bool property/////////////////
    /// fixed angle resolution
    bool b_optvalue = false;
    node->declare_parameter<bool>("fixed_resolution", true);
    node->get_parameter("fixed_resolution", b_optvalue);
    laser.setlidaropt(LidarPropFixedResolution, &b_optvalue, sizeof(bool));
    /// rotate 180
    b_optvalue = true;
    node->declare_parameter<bool>("reversion", false);
    node->get_parameter("reversion", b_optvalue);
    laser.setlidaropt(LidarPropReversion, &b_optvalue, sizeof(bool));
    /// Counterclockwise
    b_optvalue = true;
    node->declare_parameter<bool>("inverted", false);
    node->get_parameter("inverted", b_optvalue);
    laser.setlidaropt(LidarPropInverted, &b_optvalue, sizeof(bool));
    b_optvalue = true;
    node->declare_parameter<bool>("auto_reconnect", true);
    node->get_parameter("auto_reconnect", b_optvalue);
    laser.setlidaropt(LidarPropAutoReconnect, &b_optvalue, sizeof(bool));
    /// one-way communication
    b_optvalue = false;
    node->declare_parameter<bool>("isSingleChannel", true);
    node->get_parameter("isSingleChannel", b_optvalue);
    laser.setlidaropt(LidarPropSingleChannel, &b_optvalue, sizeof(bool));
    /// intensity
    b_optvalue = false;
    node->declare_parameter<bool>("intensity", false);
    node->get_parameter("intensity", b_optvalue);
    laser.setlidaropt(LidarPropIntenstiy, &b_optvalue, sizeof(bool));
    /// Motor DTR
    b_optvalue = false;
    node->declare_parameter<bool>("support_motor_dtr", true);
    node->get_parameter("support_motor_dtr", b_optvalue);
    laser.setlidaropt(LidarPropSupportMotorDtrCtrl, &b_optvalue, sizeof(bool));

    //////////////////////float property/////////////////
    /// unit: °
    float f_optvalue = 180.0f;
    node->declare_parameter<float>("angle_max", 180.0);
    node->get_parameter("angle_max", f_optvalue);
    laser.setlidaropt(LidarPropMaxAngle, &f_optvalue, sizeof(float));
    f_optvalue = -180.0f;
    node->declare_parameter<float>("angle_min", -180.0);
    node->get_parameter("angle_min", f_optvalue);
    laser.setlidaropt(LidarPropMinAngle, &f_optvalue, sizeof(float));
    /// unit: m
    f_optvalue = 64.f;
    node->declare_parameter<float>("range_max", 12.0);
    node->get_parameter("range_max", f_optvalue);
    laser.setlidaropt(LidarPropMaxRange, &f_optvalue, sizeof(float));
    f_optvalue = 0.1f;
    node->declare_parameter<float>("range_min", 0.1);
    node->get_parameter("range_min", f_optvalue);
    laser.setlidaropt(LidarPropMinRange, &f_optvalue, sizeof(float));
    /// unit: Hz
    f_optvalue = 10.f;
    node->declare_parameter<float>("frequency", 8.0);
    node->get_parameter("frequency", f_optvalue);
    laser.setlidaropt(LidarPropScanFrequency, &f_optvalue, sizeof(float));

    bool invalid_range_is_inf = false;
    node->declare_parameter<bool>("invalid_range_is_inf", false);
    node->get_parameter("invalid_range_is_inf", invalid_range_is_inf);
    laser.setlidaropt(LidarPropScanFrequency, &f_optvalue, sizeof(float));
    ```

    :::

    また、今回はLiDARにYDLiDAR X2を使用するため、ノードの起動に必要な設定ファイル`X2.yaml`を使います。
    そのため、ydlidar_rosパッケージ内にある/launch/ydlidar_launch.pyを修正します。

    :::details ydlidar_launch.pyの修正

    ```python:ydlidar_launch.py
    #!/usr/bin/python3
    # Copyright 2020, EAIBOT
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.

    from ament_index_python.packages import get_package_share_directory

    from launch import LaunchDescription
    from launch_ros.actions import LifecycleNode
    from launch_ros.actions import Node
    from launch.actions import DeclareLaunchArgument
    from launch.substitutions import LaunchConfiguration
    from launch.actions import LogInfo

    import lifecycle_msgs.msg
    import os


    def generate_launch_description():
        share_dir = get_package_share_directory('ydlidar_ros2_driver')
        parameter_file = LaunchConfiguration('params_file')
        node_name = 'ydlidar_ros2_driver_node'

        params_declare = DeclareLaunchArgument('params_file',
                                            default_value=os.path.join(
                                                share_dir, 'params', 'X2.yaml'),
                                            description='FPath to the ROS2 parameters file to use.')

        driver_node = LifecycleNode(package='ydlidar_ros2_driver',
                                    executable='ydlidar_ros2_driver_node',
                                    name='ydlidar_ros2_driver_node',
                                    output='screen',
                                    emulate_tty=True,
                                    parameters=[parameter_file],
                                    node_namespace='/',
                                    )
        tf2_node = Node(package='tf2_ros',
                        executable='static_transform_publisher',
                        name='static_tf_pub_laser',
                        arguments=['0', '0', '0.02','0', '0', '0', '1','base_link','laser_frame'],
                        )

        return LaunchDescription([
            params_declare,
            driver_node,
            tf2_node,
        ])
    
    ```

    :::

6. **YDLiDARのデバイス名固定**

    次にYDLiDARのデバイス名を固定します。デバイス名は接続ポートやタイミングによって変わってしまうので、固定しておく必要があります。
    `/etc/udev/rules.d/99-usb-serial`を開き、下記の内容を記述します。ファイルがない場合は作成します。

    ```file
    KERNEL=="ttyUSB[0-9]*", ATTRS{idVendor}=="10c4", ATTRS{idProduct}=="ea60", SYMLINK+="ttyUSB0"
    ```

    ```bash:bash
    # ビルド 
    cd ~/ros2_ws
    colcon build --symlink-install  --cmake-clean-cache  --parallel-workers 2
    ```

7. **ROS_DOMAIN_IDの設定**  

    次にROS_DOMAIN_IDを設定します。

   ```bash:bash
    # .bashrcに追記
    echo "export ROS_DOMAIN_ID=0" >> ~/.bashrc
    source ~/.bashrc
    # ドメインIDを確認
    echo $ROS_DOMAIN_ID
    ```

    :::message

    ROS_DOMAIN_IDとは、ROS2のノード同士が通信する範囲（ドメイン）を分けるための識別子です。

    同じネットワーク上に複数のロボットやシステムがあるときに、お互いの通信が干渉しないようにROS_DOMAIN_IDを設定します。

    :::

### ノートPCの環境構築

次にRaspberry Piと通信する用のノートPCを準備します。
ROS2 HumbleのインストールやROS_DOMAIN_IDの設定方法は前述したRaspberry Piのセットアップと同じであることから詳細な手順については割愛します。

- OS(Ubuntu22.04)のインストール
- ROS2 Humbleのインストール
- 環境変数の設定.bashrc　ROS_DOMAIN_IDをRaspberry Piと合わせておく
- その他必要なパッケージ、ソフトウェアのインストール

## SLAM（ロボットの自己位置推定と地図作成）

### SLAMの準備

次にSLAMを使用する手順について説明します。今回はROS向けに開発された2D SLAM用のパッケージである「slam-toolbox」を使用します。SLAM時はRviz2が起動しますが、負荷が大きいため、ノートPCからRaspberry PiにSSH接続して実行します。

1. **slam-toolboxのインストール**

    ```bash:bash
    # slam-toolboxのインストール
    sudo apt install ros-humble-slam-toolbox
    ```

2. **Rviz2の起動確認**
次に下記のコマンドを実行し、Rviz2の起動確認を行います。

   ```bash:bash
   sudo chmod 777 /dev/ttyUSB0
   cd ~/ros2_ws
   . install/setup.bash
   ros2 launch lightrover_ros lightrover_slam.launch.py
   ```

    - 確認する項目
        - LiDARが低速回転から高速回転になること。

        - ノートPC上でRviz2が起動すること。

        - LiDARによって障害物が橙色の点群で検出されていること。

    :::message alert

    LiDARが認識しない場合は、LiDARのデバイス名の設定やUSBケーブルの確認などを確認してください。

    この時点ではRviz2は起動しますが、マップの作成はされません。

    :::

3. **SLAMに必要なサービスの起動**  
    一旦、Rviz2を停止し、SLAMとモータの駆動に必要なサービスを起動する準備をします。
    lightrover_slam.launch.pyを下記の内容に修正します。他のサービスの起動をローンチファイルに記述すれば、複数台端末を起動する必要がなくなります。修正後、ビルドして再度Rviz2を起動します。

    :::details lightrover_slam.launch.py

    ```python:lightrover_slam.launch.py

    from launch import LaunchDescription
    from launch_ros.actions import LifecycleNode
    from launch_ros.actions import Node
    from launch.actions import DeclareLaunchArgument
    from launch.substitutions import LaunchConfiguration
    from launch.actions import LogInfo

    import lifecycle_msgs.msg
    import os

    def generate_launch_description():
        share_dir = get_package_share_directory('ydlidar_ros2_driver')
        rviz_config_file = os.path.join(share_dir, 'config','ydlidar.rviz')
        parameter_file = LaunchConfiguration('params_file')
        node_name = 'ydlidar_ros2_driver_node'

        static_tf_base_to_footprint = Node(package='tf2_ros',
                                        executable='static_transform_publisher',
                                        output='screen',
                                        arguments=['0', '0', '0', '3.1415', '0', '0', 'base_footprint', 'base_link'])

        params_declare = DeclareLaunchArgument('params_file',
                                            default_value=os.path.join(
                                                share_dir, 'params', 'X2.yaml'),
                                            description='FPath to the ROS2 parameters file to use.')

        
        driver_node = LifecycleNode(package='ydlidar_ros2_driver',
                                    executable='ydlidar_ros2_driver_node',
                                    name='ydlidar_ros2_driver_node',
                                    output='screen',
                                    emulate_tty=True,
                                    parameters=[parameter_file],
                                    namespace='/',
                                    )
                                    
        tf2_node = Node(package='tf2_ros',
                        executable='static_transform_publisher',output='screen',
                        arguments=['0.042', '0', '0.1094', '-1.5708', '0','0','base_link','laser_frame'],
                        )
        
        slam_node = Node(
            package='slam_toolbox', executable='sync_slam_toolbox_node',
            output='screen',
            parameters=[
                get_package_share_directory(
                    'lightrover_ros')
                + '/configuration_files/mapper_params_offline.yaml'
            ],
        )
        
        rviz2_node = Node(package='rviz2',
                        executable='rviz2',
                        name='rviz2',
                        arguments=[
                            '-d',
                            get_package_share_directory('lightrover_ros')
                                + '/config/gmapping.rviz'],
                        )
        

        # 追加
        i2c_control_node = Node(
            package='lightrover_ros',
            executable='i2c_controller',
            output='screen',
            name='lightrover_i2c_control',
            parameters=[],
            namespace='/'
        )

        # 追加
        odometry_node = Node(
            package='lightrover_ros',
            executable='odom_manager',
            output='screen',
            name='lightrover_odometry',
            parameters=[],
            namespace='/'
        )

        # 追加
        pos_control_node = Node(
            package='lightrover_ros',
            executable='pos_controller',
            output='screen',
            name='lightrover_pos_control',
            parameters=[],
            namespace='/'
        )

        return LaunchDescription([
            params_declare,
            driver_node,
            static_tf_base_to_footprint,
            tf2_node,
            slam_node,
            rviz2_node,
            i2c_control_node,  # 追加
            odometry_node, # 追加
            pos_control_node  # 追加
        ])
        
    ```

    :::

    :::message

    ライトローバーのSLAMにはi2c_controller, odom_manager, pos_controllerの3つのサービスの起動が必要です。

    i2c_controllerが起動していないと、odom_manager, pos_controllerは動作しないので、起動する順番に注意します。
    :::

4. **SLAMの起動確認**  

    Rvizが起動したら左のメニューから`[Displays] - [Global Options] - [Fixed Frame]`を`「map」`に設定する。 また、`[Displays] - [Map]、[LaserScan]、[TF]`の項目がそれぞれ`/map、/scan、/tf`のトピックを受け取っていることを確認します。

    **💡重要：TFツリーとは？**

    ライトローバーは車輪の回転によってエンコーダが回転量を読み取っており、そこから計算できる量を移動量（オドメトリ）と定義しています。移動量が計算できれば、ロボットが現在地点からどのくらい移動したか分かりますが、移動量はドリフトしていくので、リアルタイムでレーザスキャンが読み取った周辺状況と比較し補正することが必要となります。 TFツリーはこれらの座標系をツリー構造で繋ぎ、相対的な位置の関係として定義したものです。

    :::message alert
    TFの座標変換が適切に行われていないとマップが作成されなかったり、ローバーの自己位置推定がずれてしまう原因となります。  
    :::

    ライトローバーの場合、次のようなTFツリーになります。

    ![alt text](/images/SLAM.drawio.png)

    | TFフレーム間 | 配信元ノード  | 処理 |
    | ---- | ---- | ---- |
    | map → odom | slam_toolbox | 地図上のローバー位置の補正 |
    | odom → base_link | odometry  | オドメトリによるローバーの移動追跡 |
    | base_link → laserscan | static_transform_publisher  | ローバーとセンサの固定的な位置関係 |

    | トピック名 | メッセージ型  | 内容 |
    | ---- | ---- | ---- |
    | /map | nav_msgs/msg/OccupancyGrid | 地図の解像度、サイズ、原点などの情報|
    | /tf | tf2_msgs/msg/TFMessage | 動的なTF変換情報（フレーム間の位置関係） |
    | /tf_static | tf2_msgs/msg/TFMessage | 静的なTF変換情報（フレーム間の位置関係） |
    | /scan | sensor_msgs/msg/LaserScan | LiDARがスキャンした周辺情報　|
    | /odom | nav_msgs/msg/Odometry | ローバーの位置と速度の情報、/tfの計算に使用　|

### 地図作成～地図保存

1. **ライトローバーを動かしながらSLAM**

    ライトローバーを動かしながらSLAMする手順を説明します。pos_controllerにトピック送信してライトローバーを動かすことができます。ソースコードを確認すると、pos_controllerは/rover_twistトピックをTwist型のメッセージで受け取っています。つまり、Raspberry Piと同じROS_DOMAIN_IDの端末から/rover_twistトピックを送信すればローバーは指定した速度で動作するようになります。

    :::message
    Twist型は、ROS2における「速度（velocity）」を表現するメッセージ型でよく使用されます。3次元の並進速度を表すベクトルと3次元の角速度を表すベクトルで構造化されたメッセージのことです。

    Twistメッセージの例：
    `{linear: {x: 0.1, y: 0.0, z: 0.0}, angular: {x: 0.0, y: 0.0, z: 0.1}}`
    :::

    キーボードの入力からTwist型のメッセージをPublishするサンプルプログラムkey_input.pyを作りました。このPythonファイルをノートPC側の端末で実行します。

    :::details key_input.py

    ```python:key_input.py

        import rclpy
        from rclpy.node import Node
        from geometry_msgs.msg import Twist
        import sys, select, termios, tty

        class KeyboardTeleop(Node):

            def __init__(self):
                super().__init__('keyboard_teleop')
                self.publisher = self.create_publisher(Twist, '/rover_twist', 10)
                self.settings = termios.tcgetattr(sys.stdin)

            def run(self):
                try:
                    print("使えるキー: w/a/s/d/e | q: 終了")
                    linear_speed = 0.0
                    angular_speed = 0.0
                    while True:
                        key = self.get_key()
                        twist = Twist()
                        if key == 'w':
                            linear_speed  += 0.02
                            twist.linear.x = linear_speed 
                        elif key == 's':
                            linear_speed  -= 0.02
                            twist.linear.x = linear_speed 
                        elif key == 'a':
                            angular_speed += 0.1
                            twist.angular.z = angular_speed
                        elif key == 'd':
                            angular_speed -= 0.1
                            twist.angular.z = angular_speed
                        elif key == 'e':
                            linear_speed  = 0.0
                            twist.linear.x = linear_speed 
                            angular_speed = 0.0
                            twist.angular.z = angular_speed
                        elif key == 'q':
                            inear_speed  = 0.0
                            twist.linear.x = linear_speed 
                            angular_speed = 0.0
                            twist.angular.z = angular_speed
                            break
                        else:
                            continue
                        self.publisher.publish(twist)
                finally:
                    termios.tcsetattr(sys.stdin, termios.TCSADRAIN, self.settings)

            def get_key(self):
                tty.setraw(sys.stdin.fileno())
                rlist, _, _ = select.select([sys.stdin], [], [], 0.1)
                if rlist:
                    key = sys.stdin.read(1)
                else:
                    key = ''
                termios.tcsetattr(sys.stdin, termios.TCSADRAIN, self.settings)
                return key

        def main():
            rclpy.init()
            node = KeyboardTeleop()
            node.run()
            node.destroy_node()
            rclpy.shutdown()

        if __name__ == '__main__':
            main()

    ```

    :::

    キーボード操作によってローバーが走行し、地図が生成されていることを確認してください。
    LiDARがリアルタイムで検出した障害物は橙色で表示されます。地図更新時に障害物があるエリアは黒色、障害物が存在しないエリアは灰色で保存されます。

    ![alt text](/images/SLAM_complete.png)

    :::message alert

    Rviz2に表示されているLiDARの検出点群がロボットの旋回とともに大きく回転してしまい、地図生成が乱れる場合はオドメトリの計算が間違っているかTF(座標変換)の値が間違っている可能性があります。

    また、急旋回、急加速すると地図が乱れる原因となります。

    :::

2. **地図の保存**  

    地図の生成が完了すれば、Raspberry PiにSSH接続した端末を追加起動し、下記のコマンドで生成した地図を保存してください。

    ```bash:bash
    ros2 run nav2_map_server map_saver_cli -f ~/ros2_ws/src/lightrover_ros2/lightrover_navigation/maps/任意のファイル名
    ```

## Nav2による自律走行

### **Nav2について**  

Navigation2（Nav2）は、ROS2向けのロボット自律走行スタックであり、移動ロボットが安全に目標地点に到達するための一連の機能を提供するシステムツールのことです。下記に示す機能群で構成されています。

| 機能名 | 機能内容 |
| ---- | ---- |
| bt_navigator | ロボットの行動を制御するためのBehaviorTreeベースのナビゲータ。|
| planner_server |目標地点までの経路を計画する。|
| controller_server |計画された経路に沿ってロボットを制御する。|
| recovery_server | 障害が発生した際の復帰動作を計算する。|
| waypoint_follower |ウェイポイントを順次追従する。|
| costmap_2d |障害物情報を基にコストマップを生成する。|
| map_server |地図データ（.pgm と .yaml）を提供する。|
| amcl |地図上での自己位置推定を行う。|

### Nav2のセットアップ

ノートPC上でRaspberry PiにSSH接続した端末を用意する。

1. **Nav2のインストール**

    下記のコマンドを入力し、Nav2に依存しているライブラリとNav2をインストールします。

    ```bash:bash
    sudo apt install ros-humble-joint-state-publisher ros-humble-xacro
    sudo apt install ros-humble-navigation2
    sudo apt install ros-humble-nav2-bringup
    ```

2. **作成した地図ファイルの移動**

    `~/ros2_ws/src/lightrover_ros2/lightrover_navigation/maps`ディレクトリに保存した地図のpgmファイルとyamlファイルがあることを確認する。ない場合は、保存した地図のpgmファイルとyamlファイルをディレクトリに移動する。

3. **launchファイルの編集**

    `~/ros2_ws/src/lightrover_ros2/lightrover_navigation/launch`のディレクトリにある
    `lightrover_navigation.launch.py`内の下記の行を探し、保存した地図ファイル名に合わせて「ファイル名.yaml」の部分を変更する。

   ```python:lightrover_navigation.launch.py
    default = os.path.join(
                get_package_share_directory('lightrover_navigation'),
                'maps',
                'ファイル名.yaml')
   ```

4. **Nav2に必要なサービスの起動**

    端末を3台同時に起動し、下記のコマンドで各ローンチファイルを起動する。

    ```bash:bash1
    # i2c_controller, odom_manager, pos_controllerを起動するローンチファイル
    ros2 launch lightrover_ros nav_base.launch.py
    ```

    ```bash:bash2  
    # ロボットのURDFモデルを基にTF（座標変換）情報をパブリッシュする
    # lightrover_view.launch.pyについては、Rvizを起動しないように修正する
    ros2 launch lightrover_description lightrover_view.launch.py
    ```

    ```bash:bash3
    # Nav2を起動
    ros2 launch lightrover_navigation lightrover_navigation.launch.py
    ```

5. **Nav2の開始**
    1. 初期位置の設定  
        Rviz2画面上部の「2D Pose Estimate」ボタンをクリックしてから、地図上をクリックして初期位置を決めます。また、クリック後に表示される矢印の向きによってローバーの初期姿勢が決めることができます。

    2. コストマップの表示
        初期位置と姿勢を設定すると、コストマップと呼ばれる青色や紫色のエリアが地図上に表示されます。

        ![alt text](/images/Nav2_costmap.png)

        :::message

        コストマップは、ロボットの周囲の環境を2Dグリッドとして表現し、各セルに移動コストを設定したものです。移動コストは、ロボットがそのセルを通過する際の「難易度」や「危険度」を示しており、移動コストが高いとそのエリアを避けるような行動が選択されやすくなります。

        コストマップにはグローバルコストマップとローカルコストマップがあります。
        - グローバルコストマップ:　　
        生成した地図から作成される静的なコストマップ

        - ローカルコストマップ：  
        周囲の環境をリアルタイムで反映して作成される動的なコストマップ

        :::

    3. 目標地点の設定  
        初期位置の設定を完了すれば、画面上部の「2D Nav Goal」ボタンをクリックしてから、地図上をクリックしてゴール地点を決めます。また、クリック後に表示される矢印の向きによってローバーの最終的な姿勢を決めることができます。

        :::message alert
        ボタンクリックでライトローバーは即座に走行を開始するので、注意してください。
        :::

    4. ローバーの自律走行
        Rviz2画面上部の「2D Nav Goal」でゴール地点を決めれば、ライトローバーが走行し始めます。ナビゲーション中は、Rviz2画面の左下にある`Navigation2 - Feedback`が下記の状態になります。
        - `active`の場合、ローバーがゴール地点に向かっており、自律走行中を意味します。
        - `reached`の場合、ゴール地点に到着したことを意味します。
        - `aborted`の場合、失敗して中断となっています。障害物に接触したなどが原因として挙げられます。

        :::message
        `reached`になったときのRviz上の画面のローバーの位置・姿勢と現実のローバーの位置・姿勢が一致しているか確認してください。
        :::

6. **Nav2の調整**  
    `~/ros2_ws/src/lightrover_ros2/lightrover_navigation/config/nav2_params.yaml`に記述しているパラメータを変更することで、コストマップの調整や障害物周辺の回避動作の調整、ゴールへ向かう挙動の調整、ゴール地点の閾値の設定などができます。必要に応じて変更してみてください。

## まとめ

本記事では、ROS2を使用した自律走行に関する知識とライトローバー実機での自律走行の実装方法について紹介しました。また、SLAMやNavigationのような高度な技術についても必要な知識や機能の内容についてもまとめてますので、自律走行に興味がある人やROS初学者の方の学習の参考になれば幸いです。

## 参考リンク・文献

[ROS公式Wiki（日本語）](https://wiki.ros.org/ja)

[ライトローバーWebDoc](https://vstoneofficial.github.io/lightrover_webdoc/)

[ROS2とPythonで作って学ぶAIロボット入門 (KS理工学専門書)](https://www.amazon.co.jp/ROS2%E3%81%A8Python%E3%81%A7%E4%BD%9C%E3%81%A3%E3%81%A6%E5%AD%A6%E3%81%B6AI%E3%83%AD%E3%83%9C%E3%83%83%E3%83%88%E5%85%A5%E9%96%80-KS%E7%90%86%E5%B7%A5%E5%AD%A6%E5%B0%82%E9%96%80%E6%9B%B8-%E5%87%BA%E6%9D%91-%E5%85%AC%E6%88%90/dp/4065289564?tag=googhydr-22&source=dsa&hvcampaign=books&gad_source=1)

