# ouster-driver-ros
This repository contains a containerized ROS1 driver for the Ouster LiDAR. 

## Prerequisites
Instructions for installing Docker can be found [here](https://docs.docker.com/engine/install/debian/).

## Building Docker container
To create the Docker image, run:
```bash
make build
```

## Running Docker container
To run the Docker image, run:
```bash
make run
```


## Usage
### Launching Nodes
The package supports three modes of interaction, you can connect to a _live sensor_, _replay_ a
recorded bag or _record_ a new bag file using the corresponding launch files. Recently, we have
added a new mode that supports multicast. The commands are listed below:

#### Sensor Mode
The driver offers two launch files to connect to an Ouster sensor: `sensor.launch` and
`driver.launch`; they differ in terms of how the processing of incoming packets is performed.
`sensor.launch` spawns three nodelets, one to connect to the sensor and publishes raw packets to
the two other nodelets which handles converting them into **Imu**, **Image** and **PointCloud2**
messages. Meanwhile, `driver.launch` file spawn a single nodelet that handles all of these tasks.
You can invoke the two files in the same way. The following line shows how to run the node using
`driver.launch`:
```bash
roslaunch ouster_ros driver.launch      \
    sensor_hostname:=<sensor host name> \
    metadata:=<json file name>          # optional
```
`driver.launch` offers better performance and reduced overhead on the ROS bus, thus it is preferred
over `sensor.launch`. `sensor.launch` is mainly provided for backward compatibilty.

> **Note**:
> If you observe parts of the scan missing is missing, this suggests having a lots of dropped packets
> It is recommended that you increase the maximum allowed size for receive memory buffers in network
> subsystem, you can do so by running the script `network-configure.bash` under `util`.


#### Recording Mode
> Note
> As of package version 8.1, specifiying metadata file is optional since the introduction of the
> metadata topic
```bash
roslaunch ouster_ros record.launch      \
    sensor_hostname:=<sensor host name> \
    bag_file:=<optional bag file name>  \
    metadata:=<json file name>          # optional
```
#### Replay Mode
> Note
> As of package version 8.1, specifiying metadata file is optional if the bag file being replayed
> already contains the metadata topic

```bash
roslaunch ouster_ros replay.launch      \
    bag_file:=<path to rosbag file>     \
    metadata:=<json file name>          # optional if bag file has /metadata topic
```

##### PCAP Replay Mode
> Note
> To use this feature you need to compile the driver with `BUILD_PCAP` option enabled

```bash
roslaunch ouster_ros replay_pcap.launch     \
    pcap_file:=<path to ouster pcap file>   \
    metadata:=<json file name>              # required
```


#### Multicast Mode (experimental)
The multicast launch mode supports configuring the sensor to broadcast lidar packets from the same
sensor (live) to multiple active clients. You initiate this mode by using `sensor_mtp.launch` file
to start the node. You will need to specify a valid multicast group for the **udp_dest** argument
which the sensor is going to broadcast data to it. You will also need to set **mtp_main** argument
to **true**, this is need to configure the sensor with the specified **udp_dest** and any other
sensor settings. You can control on which ip (IP4 only) you wish to receive the data on this machine
from the multicast group using the **mtp_dest** argument
follows:
```bash
roslaunch ouster_ros sensor_mtp.launch      \
    sensor_hostname:=<sensor host name>     \
    udp_dest:=<multicast group ip (ipv4)>   \
    mtp_main:=true                          \
    mtp_dest:=<client ip to receive data>   # mtp_dest is optional
```
Using a different machine that belongs to the same netwok subnet, you can start another instance of
the client to start receiving sensor messages through the multicast group as shown below (note that
**mtp_main** is set to **false**):
```bash
roslaunch ouster_ros sensor_mtp.launch      \
    sensor_hostname:=<sensor host name>     \
    udp_dest:=<multicast group ip (ipv4)>   \
    mtp_main:=false                         \
    mtp_dest:=<client ip to receive data>   # mtp_dest is optional
```

> **Note:** 
> In both cases the **mtp_dest** is optional and if left unset the client will utilize the first
available interface.

### Launch Files Arguments
Each of the previously mentioned launch files include a variety of launch arguments that helps the
user customize the driver behaivor. To view the arguments that each launch file provides and their
purpose pass `--ros-args` along with the specific launch file that you are interested in. For
example, to view launche arguments of the `driver.launch` use the following command:
```bash
roslaunch ouster_ros driver.launch --ros-args
```
The command should list all available arguments, whether they are optional or required and the
description and posible values of each argument.

New launch file parameter:
**point_type**: This parameter allows to customize the point cloud that the
  driver produces through its `/ouster/points` topics. Choose one of the following
  values:
  - `original`: This uses the original point representation `ouster_ros::Point`
           of the ouster-ros driver.
  - `native`: directly maps all fields as published by the sensor to an
           equivalent point cloud representation with the additon of ring
           and timestamp fields.
  - `xyz`: the simplest point type, only has {x, y, z}
  - `xyzi`: same as xyz point type but adds intensity (signal) field. this
           type is not compatible with the low data profile.
  - `xyzir`: same as xyzi type but adds ring (channel) field.
          this type is same as Velodyne point cloud type
          this type is not compatible with the low data profile.

### Invoking Services
To execute any of the following service, first you need to open a new terminal
and source the castkin workspace again by running the command:
`source catkin_ws/devel/setup.bash` 
#### GetMetadata
To get metadata while connected to a live sensor or during a replay session invoke
the following command:
```bash
rosservice call /ouster/get_metadata
```

#### GetConfig
To get the current config of a live sensor, invoke the command:
```bash
rosservice call /ouster/get_config
```

#### SetConfig
To change config via a file while connected to a live sensor, invoke the command:
```bash
rosservice call /ouster/set_config "config_file: '<path to sensor config>'"
```

> **Note**
> Changing settings is not yet fully support during a reset operation (more on this)
  

For further detailed instructions refer to the [main guide](./docs/index.rst)


## License
[License File](./LICENSE)
