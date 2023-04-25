# chrome-build-instructions

Instructions for building Chromium (104) 
Compatible with Amazon Linux 2 (ARM64)

- Spin up an AWS EC2 machine
    - AmiID: ami-0abe92d15a280b758 in eu-west-1
    - Architecture: 64-bit (Arm)
    - Instance type: c7g.8xlarge (or larger)
    - Storage: at least 100GiB (gp3)
    - Configure an SSH key pair so you can login via SSH
- Access the build instance
    - Login via SSH
    - sudo -i
- Install build dependencies
    - yum install -y "@Development Tools" alsa-lib-devel atk-devel bc bluez-libs-devel brlapi-devel bzip2-devel cairo-devel cups-devel dbus-devel dbus-glib-devel dbus-x11 expat-devel glibc.i686 glibc-langpack-en gperf gtk3-devel httpd java-11-openjdk-devel libatomic libcap-devel libjpeg-devel libstdc++.i686 libXScrnSaver-devel libxkbcommon-x11-devel mod_ssl ncurses-compat-libs nspr-devel nss-devel pam-devel pciutils-devel perl php php-cli pulseaudio-libs-devel python python-psutil python-setuptools ruby xorg-x11-server-Xvfb zlib.i686 libcurl-devel libxml2-devel libstdc++10-devel clang
    - amazon-linux-extras enable python3.8
    - yum install python3.8 -y
    - rm -f /bin/python3
    - ln -s /bin/python3.8 /bin/python3
- Install depot_tools
    - export DEPOT_TOOLS_BOOTSTRAP_PYTHON3=0
    - cd /root
    - git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
    - export PATH="$PATH:${HOME}/depot_tools"
- Get Chromium source
    - mkdir /root/chromium && cd /root/chromium
    - git clone --depth 1 --no-tags -n https://github.com/chromium/chromium.git src
    - Copy commit SHA that you want to build from Chromium’s git repo. 
        Example: use 2e157e65025be85201e77efdbe805d43000267e3 for Chromium 104.0.5112.64
        Example 2: use 9e43919c12e51096adbb78baccb8e1e7fd5c0345 for Chromium 112.0.5615.172
    - cd src
    - git fetch --prune --depth=1 --no-tags origin 9e43919c12e51096adbb78baccb8e1e7fd5c0345
    - git checkout --quiet 9e43919c12e51096adbb78baccb8e1e7fd5c0345
- Get matching depot_tools version
    - cd /root/chromium/src
    - COMMIT_DATE=$(git log -n 1 --pretty=format:%ci)
    - cd /root/depot_tools
    - git checkout $(git rev-list -n 1 --before="$COMMIT_DATE" main)
    - export DEPOT_TOOLS_UPDATE=0
- Sync dependencies
    - touch /root/chromium/.gclient
    - Copy contents of included .gclient to /root/chromium/.gclient
    - cd /root/chromium/src
- Fix Python3 requiring too high of a GLIBC version
    - rm /root/depot_tools/.cipd_bin/3.8/bin/python3
    - ln -s /bin/python3 /root/depot_tools/.cipd_bin/3.8/bin/python3
- Add "    'condition': 'host_os == "win"', to DEPS file in reclient section" to fix missing binary for aarch64
- Change `  git_args = ['log', '-1', '--format=%H %ct']` to `  git_args = ['log', '-1', '--format="%H %ct"']` in `build/util/lastchange.py`
    - gclient sync -D --no-history --shallow --force --reset
    - gclient runhooks
- Replace NodeJs with aarch64 version
    - cd /root/chromium/src
    - sed -i 's@update_unix "darwin-x64" "mac"@# update_unix "darwin-x64" "mac"@g' third_party/node/update_node_binaries
    - sed -i 's@update_unix "darwin-arm64" "mac"@# update_unix "darwin-arm64" "mac"@g' third_party/node/update_node_binaries
    - sed -i 's@update_unix "linux-x64" "linux"@update_unix "linux-arm64" "linux"@g' third_party/node/update_node_binaries
    - ./third_party/node/update_node_binaries
    - rm -rf third_party/node/linux/node-linux-x64
    - ln -s /root/chromium/src/third_party/node/linux/node-linux-arm64 /root/chromium/src/third_party/node/linux/node-linux-x64
- Replace Java with aarch64 version
    - cd /root
    - rm -rf /root/chromium/src/third_party/jdk/current
    - wget https://github.com/adoptium/temurin11-binaries/releases/download/jdk-11.0.15%2B10/OpenJDK11U-jre_aarch64_linux_hotspot_11.0.15_10.tar.gz
    - tar zxvf OpenJDK11U-jre_aarch64_linux_hotspot_11.0.15_10.tar.gz
    - mv jdk-11.0.15+10-jre /root/chromium/src/third_party/jdk/current
- Install cmake
    - cd /root
    - wget https://cmake.org/files/v3.23/cmake-3.23.0.tar.gz
    - tar -xvzf cmake-3.23.0.tar.gz
    - cd cmake-3.23.0
    - ./bootstrap
    - make -j$(nproc)
    - make install
    - export PATH="$PATH:/usr/local/bin"
- Replace Ninja with aarch64 version
    - cd /root
    - git clone https://github.com/ninja-build/ninja.git -b v1.8.2
    - cd ninja
    - ./configure.py --bootstrap
    - rm -f /root/depot_tools/ninja
    - ln -s /root/ninja/ninja /root/depot_tools/ninja
- Install LLVM
    - cd /root/chromium/src
    - sed -i "s#dirs.lib_dir, 'libxml2.a'#os.path.join(dirs.install_dir, 'lib64'), 'libxml2.a'#g" tools/clang/scripts/build.py # UPDATED
    - Delete "      '-DLLVM_ENABLE_LLD=ON'," from same script
    - Change ./tools/clang/scripts/build.py and remove   "if args.with_ml_inliner_model" block:
    - export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/lib64
    - ./tools/clang/scripts/build.py --without-android --without-fuchsia --use-system-cmake --host-cc /bin/clang --host-cxx /bin/clang++

- Create build dir
    - mkdir -p /root/chromium/src/out/Headless
    - mount --types tmpfs --options size=48G,nr_inodes=128k,mode=1777 tmpfs /root/chromium/src/out/Headless
    - touch /root/chromium/src/out/Headless/args.gn
    - Copy contents of included args.gn to /root/chromium/src/out/Headless/args.gn
- Apply compatibility patches
    - echo '#ifndef F_LINUX_SPECIFIC_BASE' >> /usr/include/fcntl.h
    - echo '#define F_LINUX_SPECIFIC_BASE 1024' >> /usr/include/fcntl.h
    - echo '#endif' >> /usr/include/fcntl.h
    - echo '#define F_ADD_SEALS     (F_LINUX_SPECIFIC_BASE + 9)' >> /usr/include/fcntl.h
    - echo '#define F_GET_SEALS     (F_LINUX_SPECIFIC_BASE + 10)' >> /usr/include/fcntl.h
    - echo '#define F_SEAL_SEAL     0x0001' >> /usr/include/fcntl.h
    - echo '#define F_SEAL_SHRINK   0x0002' >> /usr/include/fcntl.h
    - echo '#define F_SEAL_GROW     0x0004' >> /usr/include/fcntl.h
    - echo '#define F_SEAL_WRITE    0x0008' >> /usr/include/fcntl.h
    - echo '#define F_SEAL_FUTURE_WRITE     0x0010' >> /usr/include/fcntl.h
    - sed -i '1i#define MFD_CLOEXEC             0x0001U' /root/chromium/src/v8/src/base/platform/platform-posix.cc
- Compile Chromium
    - cd /root/chromium/src
    - gn gen out/Headless
    - autoninja -C out/Headless headless_shell
- Package Chromium
    - mkdir -p /root/build/chromium/swiftshader
    - mkdir -p /root/build/chromium/lib
    - mkdir -p /root/build/chromium/fonts
    - cd /root/chromium/src
    - strip -o /root/build/chromium/chromium out/Headless/headless_shell
    - strip -o /root/build/chromium/libEGL.so out/Headless/libEGL.so
    - strip -o /root/build/chromium/libGLESv2.so out/Headless/libGLESv2.so
    - strip -o /root/build/chromium/libvk_swiftshader.so out/Headless/libvk_swiftshader.so
    - strip -o /root/build/chromium/libvulkan.so.1 out/Headless/libvulkan.so.1
    - cp out/Headless/vk_swiftshader_icd.json /root/build/chromium
    - cd /root/build/chromium
    - Copy included libs to /root/build/chromium/lib
    - Copy included fonts to /root/build/chromium/fonts
    - zip -r chromium.zip .
- Terminate the build instance

## Notes
- Clang on AL2 has version:
clang version 11.1.0 (Amazon Linux 2 11.1.0-1.amzn2.0.2)
- Clang++ same version
- AL2 binutils: binutils.aarch64                                                                                                                                                                      2.29.1-31.amzn2
- Image has kernel version 5.10, but lambda function has v4. Maybe try another image? https://eu-west-1.console.aws.amazon.com/ec2/home?region=eu-west-1#Images:visibility=public-images;search=:amazon/amzn2-ami-ecs-hvm-2.0;v=3;$case=tags:false%5C,client:false;$regex=tags:false%5C,client:false


## Attempting with ami-09ca4fd95e59ee59a (Amazon Linux 2023)

```
sudo -i
yum install python git make clang openssl-devel.aarch64 libxml2-devel.aarch64 lld libdrm-devel.aarch64 libxkbcommon-devel.aarch64 nss-devel.aarch64 perl gperf.aarch64 "@Development Tools" libXcomposite-devel.aarch64 libXdamage-devel.aarch64 libXrandr-devel.aarch64 libXtst-devel.aarch64 mesa-libgbm-devel.aarch64 alsa-lib-devel.aarch64 icu.aarch64
export DEPOT_TOOLS_BOOTSTRAP_PYTHON3=0
cd /root
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH="$PATH:${HOME}/depot_tools"

mkdir /root/chromium && cd /root/chromium
git clone --depth 1 --no-tags -n https://github.com/chromium/chromium.git src
cd src
git fetch --prune --depth=1 --no-tags origin e74af94da1b9a7bbe6f0aea365b8c9b2c5e1f429 # 114.0.5731.1
git checkout --quiet e74af94da1b9a7bbe6f0aea365b8c9b2c5e1f429 # 114.0.5731.1

cd /root/chromium/src
COMMIT_DATE=$(git log -n 1 --pretty=format:%ci)
cd /root/depot_tools
git checkout $(git rev-list -n 1 --before="$COMMIT_DATE" main)
export DEPOT_TOOLS_UPDATE=0

touch /root/chromium/.gclient
Copy contents of included .gclient to /root/chromium/.gclient
cd /root/chromium/src

- Add " 'condition': 'host_os == "win"', to DEPS file in reclient section" to fix missing binary for aarch64
gclient sync -D --no-history --shallow --force --reset

cd /root/chromium/src
sed -i 's@update_unix "darwin-x64" "mac"@# update_unix "darwin-x64" "mac"@g' third_party/node/update_node_binaries
sed -i 's@update_unix "darwin-arm64" "mac"@# update_unix "darwin-arm64" "mac"@g' third_party/node/update_node_binaries
sed -i 's@update_unix "linux-x64" "linux"@update_unix "linux-arm64" "linux"@g' third_party/node/update_node_binaries
./third_party/node/update_node_binaries
rm -rf third_party/node/linux/node-linux-x64
ln -s /root/chromium/src/third_party/node/linux/node-linux-arm64 /root/chromium/src/third_party/node/linux/node-linux-x64

cd /root
wget https://cmake.org/files/v3.23/cmake-3.23.0.tar.gz
tar -xvzf cmake-3.23.0.tar.gz
cd cmake-3.23.0
./bootstrap
make -j$(nproc)
make install
export PATH="$PATH:/usr/local/bin"

cd /root
git clone https://github.com/ninja-build/ninja.git -b v1.8.2
cd ninja
./configure.py --bootstrap
rm -f /root/depot_tools/ninja
ln -s /root/ninja/ninja /root/depot_tools/ninja

sed -i "s#dirs.lib_dir, 'libxml2.a'#os.path.join(dirs.install_dir, 'lib64'), 'libxml2.a'#g" tools/clang/scripts/build.py
sed -i "s/ldflags = \[\]/ldflags = ['-lrt -lpthread']/" tools/clang/scripts/build.py
./tools/clang/scripts/build.py --without-android --without-fuchsia --use-system-cmake --host-cc /bin/clang --host-cxx /bin/clang++ --with-ml-inliner-model=''


mkdir -p /root/chromium/src/out/Headless
mount --types tmpfs --options size=48G,nr_inodes=128k,mode=1777 tmpfs /root/chromium/src/out/Headless
touch /root/chromium/src/out/Headless/args.gn
nano out/Headless/args.gn # Add from flags
- Add `use_qt = false` to args.gn as well
sed -i 's/configs += \[ "\/\/build\/config\/linux\/dri" \]/    configs += []/g' content/gpu/BUILD.gn
sed -i 's/configs += \[ "\/\/build\/config\/linux\/dri" \]/    configs += []/g' media/gpu/sandbox/BUILD.gn
export LIBRARY_PATH="/usr/lib/gcc/aarch64-amazon-linux/11:$LIBRARY_PATH"
autoninja -C out/Headless headless_shell

mkdir -p /root/build/chromium/swiftshader
mkdir -p /root/build/chromium/lib
mkdir -p /root/build/chromium/fonts

strip -o /root/build/chromium/chromium out/Headless/headless_shell

SRC_DIR="out/Headless"
DEST_DIR="/root/build/chromium"

for src_file in "${SRC_DIR}"/*.so; do
  file_name=$(basename "${src_file}")
  dest_file="${DEST_DIR}/${file_name}"
  strip -o "${dest_file}" "${src_file}"
done

zip -r chromium.zip .
cp -r /home/ec2-user/ .
```
