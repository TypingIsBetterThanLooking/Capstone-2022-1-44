# 백견이 불여일타

## 1. 프로젝트 소개
* 프로젝트 명
  * **ZNS를 이용한 리눅스 컨테이너 I/O isolation 기법**

* 개요
  * 현재 사용되는 Block Interface SSD는 컨테이너 간에 저장 공간을 분리하지 않기 때문에 I/O 간섭 발생
  * **ZNS SSD**(Zoned Namespace SSD)란?
    * NAND flash 메모리를 **zone 단위로 구별하여 사용하는 SSD로**, 용도와 사용주기가 동일한 데이터를 순차적으로 저장하기 때문에 성능 및 공간 효율이 높음

* 목표
  * <img width="692" alt="image" src="https://user-images.githubusercontent.com/68818952/195263675-01519e78-723c-4603-ac5d-c343c5cceebc.png">
  * Device Mapper는 F2FS, EXT4와 같은 **기존 파일시스템을 사용하는 application**과 **ZBD(Zone Block Device)을 연결**해준다.
  * Dm-zoned를 이용해 리눅스 컨테이너는 ZBD에 대한 I/O를 수행할 수 있지만, **컨테이너 별 zone 격리가 일어나지 않아 I/O 간섭이 발생**
  * 그러므로 **컨테이너 별 zone 분리 할당 정책을 개발해 같은 서버를 공유하는 사용자들이 향상된 성능을 경험할 수 있도록 하는 것**이 본 과제의 목표이다.
  


---
## 2. 팀 소개
 __*:muscle:백견이 불여일타 (Typing is Better Than Looking)*__   

|김동욱|채기중|김지원|
|:-:|:-:|:-:|
|<img src="https://user-images.githubusercontent.com/71700079/190619406-503a09ff-d4a9-4837-bae2-5c78c17ed60a.jpg" width="100" height="100">|<img src="https://user-images.githubusercontent.com/71700079/190619439-ebaed8d0-a1df-4a9c-88b5-89eeaeb6eba8.jpg" width="100" height="100">|<img src="https://user-images.githubusercontent.com/76808389/195282202-cbf54a16-1e43-4b13-bfa8-501a44dab77b.png" width="85" height="85">|
|201724419|201724596|201924445|
|kosa0914@naver.com|daiywlf5251@gmail.com|kjw07227@pusan.ac.kr
---
  



## 3. 구성도
### Device Mapper

![image](https://user-images.githubusercontent.com/68818952/195265800-2324af37-8609-4231-9412-247f90f8461a.png)


---
### Mapping Table
<img width="754" alt="image" src="https://user-images.githubusercontent.com/68818952/195272996-e72f5e1d-9301-4d01-a379-6e6873b5dfb2.png">
  
*  Device Mapper에서 **chunk와 zone을 매핑하기 위해 사용하는 매핑 테이블**
*  Ext4 파일시스템에서 ZBD에 I/O 요청을 가능하게 하는 device mapper에 매핑 테이블과 **cgroup을 추가하여 구현**
*  ZNS에는 순차 쓰기를 위한 buffer zone과 순차 쓰기가 수행된 data zone이 존재하며, 이 **두 유형의 zone을 매핑 테이블에 mapping / unmapping 하는 함수 구현**
*  Chunk에 새로운 **I/O가 발생할 경우 zone을 할당하고 매핑 테이블에 cgroup id와 zone id 업데이트**

---
### Zone 할당 과정
<img width="686" alt="image" src="https://user-images.githubusercontent.com/68818952/195273787-8353aa51-9229-4127-a2b1-9334df730a12.png">
  
1. I/O를 실행하기 위해 **컨테이너가 chunk에 접근**
2. Chunk에 접근하는 **컨테이너 cgroup id**로 해당 **chunk와 zone를 컨테이너가 사용 중인지 확인**
3. 컨테이너가 chunk에 해당하는 **zone을 사용 중이면 사용중인 zone을 반환**, **사용하고 있지 않으면 unmapped zone을 할당**
4. 컨테이너가 chunk에 해당하는 **zone을 더 이상 사용하지 않는 경우 매핑 테이블에서 cgroup id를 삭제한 후 zone 할당 해제**


---
## 4. 소개 및 시연 영상

[![부산대학교 정보컴퓨터공학부 소개](http://img.youtube.com/vi/zh_gQ_lmLqE/0.jpg)](https://youtu.be/zh_gQ_lmLqE)   
정보컴퓨터공학과 유튜브 채널에 업로드되면 해당 영상의 link를 연결하겠습니다.

---
## 5. 사용법

### Environment
![image](https://user-images.githubusercontent.com/68818952/195270261-bc75fb3f-7fc5-4d02-a879-84db7a41e884.png)

### Requirements
* Device mapper install: [dm-zoned](https://github.com/westerndigitalcorporation/dm-zoned-tools)


---
### 1. Clone git repo to your linux server
```bash
git clone https://github.com/TypingIsBetterThanLooking/Capstone-2022-1-44.git
```
  
  
### 2. Install the required libraries before installing the kernel
```bash
sudo apt update
sudo apt upgrade
sudo apt install libncurses5-dev flex bison libssl-dev wget
```
  
  
### 3. Gets the current Linux kernel settings file
```bash
cp /boot/config-$(uname -r) .config
```
  
  
### 4. Check the number of cpu cores on the Linux server
```bash
grep -c processor /proc/cpuinfo
```

  
### 5. Compile and install source codes
```bash
sudo oldconfig
sudo make -j32(number of cpu cores in step 4)
sudo make modules_install -j32
sudo make install -j32
```

  
### 6. Update boot images and reboot
```bash
sudo update-grub
sudo reboot
```


### 7. Apply dm-zoned device mapper
```bash
sudo zbd reset /dev/nvme0n1
sudo dmzadm --format /dev/sdb(conv) /dev/nvme0n1(seq)
sudo dmzadm --start /dev/sdb(conv) /dev/nvme0n1(seq)
```


### 8. Formatting and mounts dm-zoned
```bash
sudo mkfs.ext4 /dev/dm-0
sudo mount /dev/dm-0 /mnt/zns(mount point)
```
