---
title: Android Input子系统专题-输入路由
date: 2020-03-20 21:19:34
tags:
- Android
- Input
---

# Android Input输入路由

​	路由在我们生活中基本都有接触，而我们这里说的输入路由主要是用在多屏幕多触摸输入设备中，设计者可以为对应的屏幕指定对应的输入设备，而这功能就是所谓的输入路由。

​	从InputFlinger的学习中我们知道，在InputFlinger中没有屏幕(Screen)的概念，而我们的输入事件是通过查找对应的Viewport(视口)来发送到对应的窗口，而窗口在指定的屏幕上，也即是对应的输入事件发送到指定的屏幕中。在底层我们是不知道哪个触摸设备对应哪个屏幕的，例如当有两个HDMI设备时，那么我们哪个触摸设备对应哪个显示器呢，这是很不好去判断的，一般查找Viewport有以下4种情况：

​	1.如果有在设备路由表中定义，则根据路由表的display id进行查找，且不论是否找到，都会直接返回(找不到对应的Viewport则返回NULL)。

​	2.如果idc配置文件中有定义touch.displayId字段，则根据该字段进行匹配查找，且不论是否找到，都会直接返回(找不到对应的Viewport则返回NULL)。

​	3.根据设备类型进行查找，如果设备属于内置类型输入设备，则会匹配到内置的display，而如果属于外置输入设备，如usb触摸屏等，则会去匹配外置的display。

​	4.在第3点的以外情况，即如果是外置输入设备，但没有匹配到外置的display，则会匹配默认的内置display。

<!-- more -->

​	以上4种情况可直接参考`TouchInputMapper::findViewport`的具体实现。

​	输入路由就是针对第一种情况。具体的源码如下：

```c++
3458 std::optional<DisplayViewport> TouchInputMapper::findViewport() {
3459     if (mParameters.hasAssociatedDisplay) {
3460         const std::optional<uint8_t> displayPort = mDevice->getAssociatedDisplayPort();
3461         if (displayPort) {
3462             // Find the viewport that contains the same port
3463             std::optional<DisplayViewport> v = mConfig.getDisplayViewportByPort(*displayPort);
3464             if (!v) {
3465                 ALOGW("Input device %s should be associated with display on port %" PRIu8 ", "
3466                         "but the corresponding viewport is not found.",
3467                         getDeviceName().c_str(), *displayPort);
3468             }
3469             return v;
3470         }
		……
3501 }
    
 262     inline std::optional<uint8_t> getAssociatedDisplayPort() const {
 263         return mAssociatedDisplayPort;
 264     }
    
1067 void InputDevice::configure(nsecs_t when, const InputReaderConfiguration* config, uint32_t changes) {
1068     mSources = 0;
1069
		……
1100
1101         if (!changes || (changes & InputReaderConfiguration::CHANGE_DISPLAY_INFO)) {
1102              // In most situations, no port will be specified.
1103             mAssociatedDisplayPort = std::nullopt;
1104             // Find the display port that corresponds to the current input port.
1105             const std::string& inputPort = mIdentifier.location;
1106             if (!inputPort.empty()) {
1107                 const std::unordered_map<std::string, uint8_t>& ports = config->portAssociations;
1108                 const auto& displayPort = ports.find(inputPort);
1109                 if (displayPort != ports.end()) {
1110                     mAssociatedDisplayPort = std::make_optional(displayPort->second);
1111                 }
1112             }
1113         }
1114
		……
1120 }
```

​	从上述的代码可以看到，输入路由就是通过InputDevice的location来从portAssociations表中去查找display。portAssociations的值我们可以逆推找到，也可以从官网的介绍文档中得知 -- `/vendor/etc/input-port-associations.xml`，值的获取如下：

```java
1905     private static String[] getInputPortAssociations() {
1906         File baseDir = Environment.getVendorDirectory();
1907         File confFile = new File(baseDir, PORT_ASSOCIATIONS_PATH);
1908
1909         try {
1910             InputStream stream = new FileInputStream(confFile);
1911             List<Pair<String, String>> associations =
1912                     ConfigurationProcessor.processInputPortAssociations(stream);
1913             List<String> associationList = flatten(associations);
1914             return associationList.toArray(new String[0]);
1915         } catch (FileNotFoundException e) {
1916             // Most of the time, file will not exist, which is expected.
1917         } catch (Exception e) {
1918             Slog.e(TAG, "Could not parse '" + confFile.getAbsolutePath() + "'", e);
1919         }
1920         return new String[0];
1921     }
 88     @VisibleForTesting
 89     static List<Pair<String, String>> processInputPortAssociations(InputStream xml)
 90             throws Exception {
 91         List<Pair<String, String>> associations = new ArrayList<>();
 92         try (InputStreamReader confReader = new InputStreamReader(xml)) {
 93             XmlPullParser parser = Xml.newPullParser();
 94             parser.setInput(confReader);
 95             XmlUtils.beginDocument(parser, "ports");
 96
 97             while (true) {
 98                 XmlUtils.nextElement(parser);
 99                 String entryName = parser.getName();
100                 if (!"port".equals(entryName)) {
101                     break;
102                 }
103                 String inputPort = parser.getAttributeValue(null, "input");
104                 String displayPort = parser.getAttributeValue(null, "display");
105                 if (TextUtils.isEmpty(inputPort) || TextUtils.isEmpty(displayPort)) {
106                     // This is likely an error by an OEM during device configuration
107                     Slog.wtf(TAG, "Ignoring incomplete entry");
108                     continue;
109                 }
110                 try {
111                     Integer.parseUnsignedInt(displayPort);
112                 } catch (NumberFormatException e) {
113                     Slog.wtf(TAG, "Display port should be an integer");
114                     continue;
115                 }
116                 associations.add(new Pair<>(inputPort, displayPort));
117             }
118         }
119         return associations;
120     }
```

​	从代码上看，输入路由的配置文件input-port-associations.xml的格式如下：

```xml
<ports>
    <port display="0" input="usb-xhci-hcd.0.auto-1.1/input0" />
    <port display="1" input="usb-xhci-hcd.0.auto-1.2/input0" />
</ports>
```

​	顶层以<ports>为tag，子节点为一组display-input配置。display属性是一个整形值，对应了displayId，input属性是输入设备的描述。回顾`InputDevice::configure`方法的内容，input的字段对应了InputDevice的location字段。displayId我们很好理解，就是显示设备在Display系统中的idz中，我们可以通过`dumpsys display`来获取，而这个location值也可以通过`dumpsys input`来获取设备对应的值。display的值是一个整数，范围是[0,255]。

​	我们再看以下location这个值是从哪里来的：

```c++
1189 status_t EventHub::openDeviceLocked(const char* devicePath) {
    	……
1240     // Get device physical location.
1241     if(ioctl(fd, EVIOCGPHYS(sizeof(buffer) - 1), &buffer) < 1) {
1242         //fprintf(stderr, "could not get location for %s, %s\n", devicePath, strerror(errno));
1243     } else {
1244         buffer[sizeof(buffer) - 1] = '\0';
1245         identifier.location = buffer;
1246     }
    	……
1477 }
```

​	location就是input设备驱动中的phys。



# 总结

​	输入路由其实就是简单的由一个xml文件对输入设备(触摸设备)和显示设备之间进行绑定。我觉得可以由以下的用途：

​	1.多个外置的输入设备和显示设备，需要指定对应的输入和显示设备关系，可以使用输入路由控制。

​	2.一个内置一个外置的输入设备和显示设备。当两个显示器进行同显时(此时只有一个viewport)，那么两个输入设备都可以控制主显，而我们又不希望如此时，可以通过输入路由指定外置输入设备控制外置显示设备(即当只有异显时才生效)。

​	3.将某个输入设备的display设成一个不存在的displayId，那么就可以禁止掉输入设备的事件(有点多此一举，因为可通过`excluded-input-devices.xml`让对应的设备不进行注册，但这种做法可以禁止掉一类型的输入设备事件上报，也不失为一种方式)。

​	如果大家还有其他想到的用途，欢迎交流~

，