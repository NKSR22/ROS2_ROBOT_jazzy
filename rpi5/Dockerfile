FROM ros:jazzy-ros-base

RUN apt-get update && apt-get install -y --no-install-recommends \
    ros-jazzy-rmw-cyclonedds-cpp \
    ros-jazzy-sllidar-ros2 \
    ros-jazzy-v4l2-camera \
    v4l-utils \
    && rm -rf /var/lib/apt/lists/*

ENV RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

RUN mkdir -p /ros2_ws/src
WORKDIR /ros2_ws

CMD ["bash"]
