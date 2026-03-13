FROM osrf/ros:jazzy-desktop

RUN apt-get update && apt-get install -y \
    ros-jazzy-navigation2 \
    ros-jazzy-nav2-bringup \
    ros-jazzy-slam-toolbox \
    ros-jazzy-ros-gz \
    ros-jazzy-rmw-cyclonedds-cpp \
    nano \
    && rm -rf /var/lib/apt/lists/*

ENV RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

RUN mkdir -p /ros2_ws/src
WORKDIR /ros2_ws

CMD ["bash"]
