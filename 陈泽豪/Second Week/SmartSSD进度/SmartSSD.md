#### Record

1. 安装XRT和Depolyment installation file
2. sudo apt install ./xilinx.xxxx.deb
3. flash操作，此过程与user guide并不一致，初步猜测这与runtime的版本有关。
4. Flashed successfully之后，进行cold boot。
5. 开机自动进入bios setting，无法进入系统（不知道什么原因）。首先，BIOS setting对两个EFUI可见，但是目前并不清楚进不了系统的原因。把SmartSSD拆除之后则可顺利进入系统。我在Xilinx社区提了issue，目前还没有人提出有建设性的意见。