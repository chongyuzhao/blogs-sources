---
title: Input系统映Map射过程
date: 2020-03-14 23:30:30
tags:
---

# android Input kcm/kl 映射过程  
---
[TOC]
****
## 一、Input系统映射文件的匹配
<font size=4> 
&#8195;我们知道，输入设备在驱动层上报到上层的只有事件类型和scancode，各种不同的设备所上报的scancode是不一样的，而这些scancode应该是设备所发上来的电平值，Android Input系统为了兼容这些不同的设备，因此在Native层将这些东西进行了一次映射，将scancode映射成Android系统独一无二的值，这样，不管是按键设成字符‘A’，还是键盘输入A，都变成一个唯一值"KEYCODE_A"。
&#8195;下面我们对这个过程进行分析。 

&#8195;映射的过程缺不了映射规则，在Android系统中，映射规则就是使用映射文件进行映射，而这些映射文件就是kcm(KeyCharacterMap)和kl(KeyLayout)两种文件，这些映射文件是存放子在"/system/etc/keylayout"目录下。
&#8195;当InputReader创建一个新设备的时候，就会去匹配一个映射文件。我们以其进行分析。(以AndroidP的源码进行分析)

```java
//frameworks/native/service/inputfinger/EventHub.cpp
1110 status_t EventHub::openDeviceLocked(const char *devicePath) {
1111     char buffer[80];
1112 
1113     ALOGV("Opening device: %s", devicePath);
1114 
1115     int fd = open(devicePath, O_RDWR | O_CLOEXEC | O_NONBLOCK);
1116     if(fd < 0) {
1117         ALOGE("could not open %s, %s\n", devicePath, strerror(errno));
1118         return -1;
1119     }
1120 
1121     InputDeviceIdentifier identifier;
1122 
1123     // Get device name.
1124     if(ioctl(fd, EVIOCGNAME(sizeof(buffer) - 1), &buffer) < 1) {
1125         //fprintf(stderr, "could not get device name for %s, %s\n", devicePath, strerror(errno));
1126     } else {
1127         buffer[sizeof(buffer) - 1] = '\0';
1128         identifier.name.setTo(buffer);
1129     }
1130 
1131     // Check to see if the device is on our excluded list
1132     for (size_t i = 0; i < mExcludedDevices.size(); i++) {
1133         const String8& item = mExcludedDevices.itemAt(i);
1134         if (identifier.name == item) {
1135             ALOGI("ignoring event id %s driver %s\n", devicePath, item.string());
1136             close(fd);
1137             return -1;
1138         }
1139     }
1140
1141     // Get device driver version.
1142     int driverVersion;
1143     if(ioctl(fd, EVIOCGVERSION, &driverVersion)) {
1144         ALOGE("could not get driver version for %s, %s\n", devicePath, strerror(errno));
1145         close(fd);
1146         return -1;
1147     }
1148 
1149     // Get device identifier.
1150     struct input_id inputId;
1151     if(ioctl(fd, EVIOCGID, &inputId)) {
1152         ALOGE("could not get device input id for %s, %s\n", devicePath, strerror(errno));
1153         close(fd);
1154         return -1;
1155     }
1156     identifier.bus = inputId.bustype;
1157     identifier.product = inputId.product;
1158     identifier.vendor = inputId.vendor;
1159     identifier.version = inputId.version;
1160 
1161     // Get device physical location.
1162     if(ioctl(fd, EVIOCGPHYS(sizeof(buffer) - 1), &buffer) < 1) {
1163         //fprintf(stderr, "could not get location for %s, %s\n", devicePath, strerror(errno));
1164     } else {
1165         buffer[sizeof(buffer) - 1] = '\0';
1166         identifier.location.setTo(buffer);
1167     }
1168 
1169     // Get device unique id.
1170     if(ioctl(fd, EVIOCGUNIQ(sizeof(buffer) - 1), &buffer) < 1) {
1171         //fprintf(stderr, "could not get idstring for %s, %s\n", devicePath, strerror(errno));
1172     } else {
1173         buffer[sizeof(buffer) - 1] = '\0';
1174         identifier.uniqueId.setTo(buffer);
1175     }
1176 
1177     // Fill in the descriptor.
1178     assignDescriptorLocked(identifier);
                         ……
1198     loadConfigurationLocked(device);     //匹配映射文件
                        ……
1384     addDeviceLocked(device);
1385     return OK;
1386 }
```
&#8195;openDeviceLocked这个方法很长，前面一部分是从驱动中读取设备的信息，再封装起来。而去匹配映射文件则是在loadConfigurationLocked方法中进行。

```java
//frameworks/native/service/inputfinger/EventHub.cpp
1491 void EventHub::loadConfigurationLocked(Device* device) {
1492     device->configurationFile = getInputDeviceConfigurationFilePathByDeviceIdentifier(
1493             device->identifier, INPUT_DEVICE_CONFIGURATION_FILE_TYPE_CONFIGURATION);   //②
1494     if (device->configurationFile.isEmpty()) {
1495         ALOGD("No input device configuration file found for device '%s'.",
1496                 device->identifier.name.string());
1497     } else {
1498         status_t status = PropertyMap::load(device->configurationFile,
1499                 &device->configuration);    //加载解析
1500         if (status) {
1501             ALOGE("Error loading input device configuration file for device '%s'.  "
1502                     "Using default configuration.",
1503                     device->identifier.name.string());
1504         }
1505     }
1506 }
```
&#8195;真正去匹配的方法是getInputDeviceConfigurationFilePathByDeviceIdentifier，当匹配成功后则会由PropertyMap::load进行加载解析。
```java
//frameworks/native/libs/input/InputDevice.cpp
 57 String8 getInputDeviceConfigurationFilePathByDeviceIdentifier(
 58         const InputDeviceIdentifier& deviceIdentifier,
 59         InputDeviceConfigurationFileType type) {
 60     if (deviceIdentifier.vendor !=0 && deviceIdentifier.product != 0) {
 61         if (deviceIdentifier.version != 0) {
 62             // Try vendor product version.
 63             String8 versionPath(getInputDeviceConfigurationFilePathByName(
 64                     String8::format("Vendor_%04x_Product_%04x_Version_%04x",
 65                             deviceIdentifier.vendor, deviceIdentifier.product,
 66                             deviceIdentifier.version),
 67                     type));
 68             if (!versionPath.isEmpty()) {
 69                 return versionPath;
 70             }
 71         }
 72 
 73         // Try vendor product.
 74         String8 productPath(getInputDeviceConfigurationFilePathByName(
 75                 String8::format("Vendor_%04x_Product_%04x",
 76                         deviceIdentifier.vendor, deviceIdentifier.product),
 77                 type));
 78         if (!productPath.isEmpty()) {
 79             return productPath;
 80         }
 81     }
 82 
 83     // Try device name.
 84     return getInputDeviceConfigurationFilePathByName(deviceIdentifier.name, type);
 85 }
```

&#8195;getInputDeviceConfigurationFilePathByName就是去通过名字来匹配对应的映射文件，从上文中可以看到，映射文件的文件名有一下三种匹配方式:
&#8195;1.Vendor_{vendorId}_Product_{ProductId}_Version_{VersionId}
&#8195;2.Vendor_{VendorId}_Product_{ProductId}
&#8195;3.DeviceName

&#8195;这三种匹配方式会轮流进行查找，只有查找到一个符合的映射文件，就会退出匹配过程。下面继续看一下接下来的匹配过程。
```java
//frameworks/native/libs/input/InputDevice.cpp
 87 String8 getInputDeviceConfigurationFilePathByName(
 88         const String8& name, InputDeviceConfigurationFileType type) {
 89     // Search system repository.
 90     String8 path;
 91 
 92     // Treblized input device config files will be located /odm/usr or /vendor/usr.
 93     const char *rootsForPartition[] {"/odm", "/vendor", getenv("ANDROID_ROOT")};
 94     for (size_t i = 0; i < size(rootsForPartition); i++) {
 95         path.setTo(rootsForPartition[i]);
 96         path.append("/usr/");
 97         appendInputDeviceConfigurationFileRelativePath(path, name, type);
 98 #if DEBUG_PROBE
 99         ALOGD("Probing for system provided input device configuration file: path='%s'",
100               path.string());
101 #endif
102         if (!access(path.string(), R_OK)) {
103 #if DEBUG_PROBE
104             ALOGD("Found");
105 #endif
106             return path;
107         }
108     }
109 
110     // Search user repository.
111     // TODO Should only look here if not in safe mode.
112     path.setTo(getenv("ANDROID_DATA"));
113     path.append("/system/devices/");
114     appendInputDeviceConfigurationFileRelativePath(path, name, type);
115 #if DEBUG_PROBE
116     ALOGD("Probing for system user input device configuration file: path='%s'", path.string());
117 #endif
118     if (!access(path.string(), R_OK)) {
119 #if DEBUG_PROBE
120         ALOGD("Found");
121 #endif
122         return path;
123     }
124 
125     // Not found.
126 #if DEBUG_PROBE
127     ALOGD("Probe failed to find input device configuration file: name='%s', type=%d",
128             name.string(), type);
129 #endif
130     return String8();
131 }
```
​	映射文件会从下面几个文件夹中去查找：/odm/，/vendor/，/system/(ANDROID_ROOT在P上就是/system)，/data/system/devices/(ANDROID_DATA在P上是/data)。匹配的顺序就是优先级，也就是说，只要匹配到一个合适的文件，就不再继续进行匹配。

```java
static void appendInputDeviceConfigurationFileRelativePath(String8& path,
        const String8& name, InputDeviceConfigurationFileType type) {
    path.append(CONFIGURATION_FILE_DIR[type]);
    for (size_t i = 0; i < name.length(); i++) {
        char ch = name[i];
        if (!isValidNameChar(ch)) {
            ch = '_';
        }
        path.append(&ch, 1); 
    }   
    path.append(CONFIGURATION_FILE_EXTENSION[type]);
}
static const char* CONFIGURATION_FILE_DIR[] = { 
        "idc/",
        "keylayout/",
        "keychars/",
};

static const char* CONFIGURATION_FILE_EXTENSION[] = { 
        ".idc",
        ".kl",
        ".kcm",
};
```





</font>