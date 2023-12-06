# extend disk on ubuntu
## check logical volume and volume group status first
```
vgdisplay
```
```
lvdisplay
```
extend logical volume
```
lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
```
apply changes
```
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```
## another way for expand
```
cfdisk
```
```
pvresize /dev/sda3
```
```
pvdisplay
```
```
vgdisplay
```
```
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
```
```
lvdisplay
```
```
resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```
```
df -h
```