---
link: https://blog.csdn.net/l328873524/article/details/103051692
title: AndroidQAudioPolicyManager之初始化篇initialize
description: 阿三
keywords: AndroidQAudioPolicyManager之初始化篇initialize
author: 轻量级Lz Csdn认证博客专家 Csdn认证企业博客 码龄9年 暂无认证
date: 2020-05-31T15:29:00.000Z
publisher: null
tags:
    - CSDN转载
    - Audio
stats: paragraph=120 sentences=112, words=2193
---
关于AudioPolicyManager网上的资料很多，但大多是关于解析audio_policy_configuration.xml解析，这个确实很重要，因为AudioPolicyManager所有初始化赋值基本都是从这个xml的解析开始的，但关于这个xml解析可以结合网上一些资料以及源码自己慢慢琢磨，只有这个弄明白了，AudioPolicyManager中的东西才好继续研究，源码路径如下
frameworks/av/services/audiopolicy/common/managerdefinitions/src/Serializer.cpp，而audio_policy_configuration.xml的路径我就不贴了，因为每个厂商定制的时候这个路径都是不同的，基本也很少使用framework下的。

关于AudioPolicyManager的初始化除了解析audio_policy_configuation.xml外，还有一个initialize，我们几天就来分析下这个函数。

```cpp
    audio_policy::EngineInstance *engineInstance = audio_policy::EngineInstance::getInstance();
    if (!engineInstance) {
        ALOGE("%s:  Could not get an instance of policy engine", __FUNCTION__);
        return NO_INIT;
    }

    mEngine = engineInstance->queryInterface<AudioPolicyManagerInterface>();
    if (mEngine == NULL) {
        ALOGE("%s: Failed to get Policy Engine Interface", __FUNCTION__);
        return NO_INIT;
    }
    mEngine->setObserver(this);
    status_t status = mEngine->initCheck();
    if (status != NO_ERROR) {
        LOG_FATAL("Policy engine not initialized(err=%d)", status);
        return status;
    }
```

这一部分主要是初始化mEngine，Engine主要是处理音源路由策略的，我们继续看，接下来开始进入mHwModulesAll的for循环，mHwModulesAll是我们在解析xml的时候赋值的，具体对应xml的标签 `<module name="primary" halversion="3.0"></module>`将所有module标签解析后add到mHwModulesAll中组成mHwModulesAll。

```cpp
        hwModule->setHandle(mpClientInterface->loadHwModule(hwModule->getName()));
        if (hwModule->getHandle() == AUDIO_MODULE_HANDLE_NONE) {
            ALOGW("could not open HW module %s", hwModule->getName());
            continue;
        }
        mHwModules.push_back(hwModule);
```

我们知道hwModule在初始化的时候getHandle就是AUDIO_MODULE_HANDLE_NONE，但如果mpClientInterface->loadHwModule(hwModule->getName())成功了则会把这个返回值给到hwModule的handle。
我们先看下mpClientInterface->loadHwModule(hwModule->getName())这个过程，hwModule->getName()就是刚才说的module标签下的name属性，这里是"primary"。mpClientInterface就是AudioPolicyClientInterface，而正真继承这个interface的是AudioPolicyClient，我们找到AudioPolicyClient实现loadHwModule的地方，源码路径frameworks/av/services/audiopolicy/service/AudioPolicyClientImpl.cpp

```cpp
audio_module_handle_t AudioPolicyService::AudioPolicyClient::loadHwModule(const char *name)
{
    sp<IAudioFlinger> af = AudioSystem::get_audio_flinger();
    if (af == 0) {
        ALOGW("%s: could not get AudioFlinger", __func__);
        return AUDIO_MODULE_HANDLE_NONE;
    }

    return af->loadHwModule(name);
}
```

这个调用到了AudioFling的loadHwModule了，我们看下AudioFlinger中的逻辑

```cpp
audio_module_handle_t AudioFlinger::loadHwModule(const char *name)
{
    if (name == NULL) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    if (!settingsAllowed()) {
        return AUDIO_MODULE_HANDLE_NONE;
    }
    Mutex::Autolock _l(mLock);
    return loadHwModule_l(name);
}
```

检查了参数和权限后，继续调用了loadHwModule_l(name)

```cpp
audio_module_handle_t AudioFlinger::loadHwModule_l(const char *name)
{

    for (size_t i = 0; i < mAudioHwDevs.size(); i++) {
        if (strncmp(mAudioHwDevs.valueAt(i)->moduleName(), name, strlen(name)) == 0) {
            ALOGW("loadHwModule() module %s already loaded", name);
            return mAudioHwDevs.keyAt(i);
        }
    }

    sp<DeviceHalInterface> dev;

    int rc = mDevicesFactoryHal->openDevice(name, &dev);
    if (rc) {
        ALOGE("loadHwModule() error %d loading module %s", rc, name);
        return AUDIO_MODULE_HANDLE_NONE;
    }

    mHardwareStatus = AUDIO_HW_INIT;
    rc = dev->initCheck();
    mHardwareStatus = AUDIO_HW_IDLE;
    if (rc) {
        ALOGE("loadHwModule() init check error %d for module %s", rc, name);
        return AUDIO_MODULE_HANDLE_NONE;
    }

    AudioHwDevice::Flags flags = static_cast<AudioHwDevice::Flags>(0);
    {
        AutoMutex lock(mHardwareLock);

        if (0 == mAudioHwDevs.size()) {
            mHardwareStatus = AUDIO_HW_GET_MASTER_VOLUME;
            float mv;
            if (OK == dev->getMasterVolume(&mv)) {
                mMasterVolume = mv;
            }

            mHardwareStatus = AUDIO_HW_GET_MASTER_MUTE;
            bool mm;
            if (OK == dev->getMasterMute(&mm)) {
                mMasterMute = mm;
            }
        }

        mHardwareStatus = AUDIO_HW_SET_MASTER_VOLUME;
        if (OK == dev->setMasterVolume(mMasterVolume)) {
            flags = static_cast<AudioHwDevice::Flags>(flags |
                    AudioHwDevice::AHWD_CAN_SET_MASTER_VOLUME);
        }

        mHardwareStatus = AUDIO_HW_SET_MASTER_MUTE;
        if (OK == dev->setMasterMute(mMasterMute)) {
            flags = static_cast<AudioHwDevice::Flags>(flags |
                    AudioHwDevice::AHWD_CAN_SET_MASTER_MUTE);
        }

        mHardwareStatus = AUDIO_HW_IDLE;
    }
    if (strcmp(name, AUDIO_HARDWARE_MODULE_ID_MSD) == 0) {

        flags = static_cast<AudioHwDevice::Flags>(flags | AudioHwDevice::AHWD_IS_INSERT);
    }

    audio_module_handle_t handle = (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));

    ALOGI("loadHwModule() Loaded %s audio interface, handle %d", name, handle);

    return handle;

}
```

代码有点长，先分析下mDevicesFactoryHal->openDevice(name, &dev)，mDevicesFactoryHal是在AudioFlinger启动的时候mDevicesFactoryHal = DevicesFactoryHalInterface::create();创建的，源码路径frameworks/av/media/libaudiohal/DevicesFactoryHalInterface.cpp

```cpp
sp<DevicesFactoryHalInterface> DevicesFactoryHalInterface::create() {
    if (hardware::audio::V5_0::IDevicesFactory::getService() != nullptr) {
        return V5_0::createDevicesFactoryHal();
    }
    if (hardware::audio::V4_0::IDevicesFactory::getService() != nullptr) {
        return V4_0::createDevicesFactoryHal();
    }
    if (hardware::audio::V2_0::IDevicesFactory::getService() != nullptr) {
        return V2_0::createDevicesFactoryHal();
    }
    return nullptr;
}
```

根据不同版本创建createDevicesFactoryHal，这里暂时分析下2.0的hal。源码路径如下frameworks/av/media/libaudiohal/impl/DevicesFactoryHalHybrid.h

```cpp
sp<DevicesFactoryHalInterface> createDevicesFactoryHal() {
    return new DevicesFactoryHalHybrid();
}
```

我们继续看下new DevicesFactoryHalHybrid()

```cpp
DevicesFactoryHalHybrid::DevicesFactoryHalHybrid()
        : mLocalFactory(new DevicesFactoryHalLocal()),
          mHidlFactory(new DevicesFactoryHalHidl()) {
}
```

分别初始化了DevicesFactoryHalLocal和DevicesFactoryHalHidl，其实DevicesFactoryHalLocal主要是为了兼容老版本的，我们回到AudioFlinger继续看下mDevicesFactoryHal->openDevice(name, &dev)，这里就调到了DevicesFactoryHalHybrid的openDevice

```cpp
status_t DevicesFactoryHalHybrid::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mHidlFactory != 0 && strcmp(AUDIO_HARDWARE_MODULE_ID_A2DP, name) != 0 &&
        strcmp(AUDIO_HARDWARE_MODULE_ID_HEARING_AID, name) != 0) {
        return mHidlFactory->openDevice(name, device);
    }
    return mLocalFactory->openDevice(name, device);
}
```

这里mHidlFactory不为0，我们的name也不是a2dp,因此我们使用的是 mHidlFactory->openDevice(name, device)而对于a2dp的hwModule则会使用mLocalFactory->openDevice(name, device)。继续

```cpp
status_t DevicesFactoryHalHidl::openDevice(const char *name, sp<DeviceHalInterface> *device) {
    if (mDeviceFactories.empty()) return NO_INIT;
    status_t status;
    auto hidlId = idFromHal(name, &status);
    if (status != OK) return status;
    Result retval = Result::NOT_INITIALIZED;
    for (const auto& factory : mDeviceFactories) {
        Return<void> ret = factory->openDevice(
                hidlId,
                [&](Result r, const sp<IDevice>& result) {
                    retval = r;
                    if (retval == Result::OK) {
                        *device = new DeviceHalHidl(result);
                    }
                });
        if (!ret.isOk()) return FAILED_TRANSACTION;
        switch (retval) {

            case Result::OK: return OK;

            case Result::NOT_INITIALIZED: return NO_INIT;

            default: ;
        }
    }
    ALOGW("The specified device name is not recognized: \"%s\"", name);
    return BAD_VALUE;
}
```

我们先看下idFromHal

```cpp
static IDevicesFactory::Device idFromHal(const char *name, status_t* status) {
    *status = OK;
    if (strcmp(name, AUDIO_HARDWARE_MODULE_ID_PRIMARY) == 0) {
        return IDevicesFactory::Device::PRIMARY;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_A2DP) == 0) {
        return IDevicesFactory::Device::A2DP;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_USB) == 0) {
        return IDevicesFactory::Device::USB;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_REMOTE_SUBMIX) == 0) {
        return IDevicesFactory::Device::R_SUBMIX;
    } else if(strcmp(name, AUDIO_HARDWARE_MODULE_ID_STUB) == 0) {
        return IDevicesFactory::Device::STUB;
    }
    ALOGE("Invalid device name %s", name);
    *status = BAD_VALUE;
    return {};
}
```

继续factory->openDevice在看下

```cpp
#if MAJOR_VERSION == 2
Return<void> DevicesFactory::openDevice(IDevicesFactory::Device device, openDevice_cb _hidl_cb) {
    switch (device) {
        case IDevicesFactory::Device::PRIMARY:
            return openDevice<PrimaryDevice>(AUDIO_HARDWARE_MODULE_ID_PRIMARY, _hidl_cb);
        case IDevicesFactory::Device::A2DP:
            return openDevice(AUDIO_HARDWARE_MODULE_ID_A2DP, _hidl_cb);
        case IDevicesFactory::Device::USB:
            return openDevice(AUDIO_HARDWARE_MODULE_ID_USB, _hidl_cb);
        case IDevicesFactory::Device::R_SUBMIX:
            return openDevice(AUDIO_HARDWARE_MODULE_ID_REMOTE_SUBMIX, _hidl_cb);
        case IDevicesFactory::Device::STUB:
            return openDevice(AUDIO_HARDWARE_MODULE_ID_STUB, _hidl_cb);
    }
    _hidl_cb(Result::INVALID_ARGUMENTS, nullptr);
    return Void();
}
#elif MAJOR_VERSION >= 4
Return<void> DevicesFactory::openDevice(const hidl_string& moduleName, openDevice_cb _hidl_cb) {
    if (moduleName == AUDIO_HARDWARE_MODULE_ID_PRIMARY) {
        return openDevice<PrimaryDevice>(moduleName.c_str(), _hidl_cb);
    }
    return openDevice(moduleName.c_str(), _hidl_cb);
}
```

这里也是只分析阿2.0的逻辑，根据我们传入的primary这里直接到 openDevice(AUDIO_HARDWARE_MODULE_ID_PRIMARY, _hidl_cb);

```cpp
template <class DeviceShim, class Callback>
Return<void> DevicesFactory::openDevice(const char* moduleName, Callback _hidl_cb) {
    audio_hw_device_t* halDevice;
    Result retval(Result::INVALID_ARGUMENTS);
    sp<DeviceShim> result;
    int halStatus = loadAudioInterface(moduleName, &halDevice);
    if (halStatus == OK) {
        result = new DeviceShim(halDevice);
        retval = Result::OK;
    } else if (halStatus == -EINVAL) {
        retval = Result::NOT_INITIALIZED;
    }
    _hidl_cb(retval, result);
    return Void();
}
```

这里调用了loadAudioInterface(moduleName, &halDevice)而halDevice是hal下device的struct具体定义在
hardware/libhardware/include/hardware/audio.h中，继续看下loadAudioInterface

```cpp
int DevicesFactory::loadAudioInterface(const char* if_name, audio_hw_device_t** dev) {
    const hw_module_t* mod;
    int rc;

    rc = hw_get_module_by_class(AUDIO_HARDWARE_MODULE_ID, if_name, &mod);
    if (rc) {
        ALOGE("%s couldn't load audio hw module %s.%s (%s)", __func__, AUDIO_HARDWARE_MODULE_ID,
              if_name, strerror(-rc));
        goto out;
    }
    rc = audio_hw_device_open(mod, dev);
    if (rc) {
        ALOGE("%s couldn't open audio hw device in %s.%s (%s)", __func__, AUDIO_HARDWARE_MODULE_ID,
              if_name, strerror(-rc));
        goto out;
    }
    if ((*dev)->common.version < AUDIO_DEVICE_API_VERSION_MIN) {
        ALOGE("%s wrong audio hw device version %04x", __func__, (*dev)->common.version);
        rc = -EINVAL;
        audio_hw_device_close(*dev);
        goto out;
    }
    return OK;

out:
    *dev = NULL;
    return rc;
}
```

这里调用了audio_hw_device_open，实际这里就调用到了audio hal下了，

```cpp
static inline int audio_hw_device_open(const struct hw_module_t* module,
                                       struct audio_hw_device** device)
{
    return module->methods->open(module, AUDIO_HARDWARE_INTERFACE,
                                 TO_HW_DEVICE_T_OPEN(device));
}
```

这里的module->methods->open对应的就是audio hal下的adev_open

```cpp
static struct hw_module_methods_t hal_module_methods = {
    .open = adev_open,
};
```

接下来就是audio hal里的adev_open的初始化了，这里就暂不向下分析了。回到AudioFlinger中继续初始化masterVolume和mute，然后

```cpp
    audio_module_handle_t handle = (audio_module_handle_t) nextUniqueId(AUDIO_UNIQUE_ID_USE_MODULE);
    mAudioHwDevs.add(handle, new AudioHwDevice(handle, name, dev, flags));
```

创建handle和new AudioHwDevice然后存入mAudioHwDevs中，到此loadHwModule就结束了。继续回到AudioPolicyManager中

```cpp
mHwModules.push_back(hwModule);
```

我们之前解析是拿到的mHwModulesAll，通过AudioFlinger的loadHwModule成功后的HwModele再放入mHwModules中。

```cpp

        for (const auto& outProfile : hwModule->getOutputProfiles()) {

            if (!outProfile->canOpenNewIo()) {
                ALOGE("Invalid Output profile max open count %u for profile %s",
                      outProfile->maxOpenCount, outProfile->getTagName().c_str());
                continue;
            }

            if (!outProfile->hasSupportedDevices()) {
                ALOGW("Output profile contains no device on module %s", hwModule->getName());
                continue;
            }

            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_TTS) != 0) {
                mTtsOutputAvailable = true;
            }

            if ((outProfile->getFlags() & AUDIO_OUTPUT_FLAG_DIRECT) != 0) {
                continue;
            }

            const DeviceVector &supportedDevices = outProfile->getSupportedDevices();

            DeviceVector availProfileDevices = supportedDevices.filter(mAvailableOutputDevices);
            sp<DeviceDescriptor> supportedDevice = 0;

            if (supportedDevices.contains(mDefaultOutputDevice)) {
                supportedDevice = mDefaultOutputDevice;
            } else {

                if (availProfileDevices.isEmpty()) {
                    continue;
                }
                supportedDevice = availProfileDevices.itemAt(0);
            }
            if (!mAvailableOutputDevices.contains(supportedDevice)) {
                continue;
            }
            sp<SwAudioOutputDescriptor> outputDesc = new SwAudioOutputDescriptor(outProfile,
                                                                                 mpClientInterface);
            audio_io_handle_t output = AUDIO_IO_HANDLE_NONE;
            status_t status = outputDesc->open(nullptr, DeviceVector(supportedDevice),
                                               AUDIO_STREAM_DEFAULT,
                                               AUDIO_OUTPUT_FLAG_NONE, &output);
            if (status != NO_ERROR) {
                ALOGW("Cannot open output stream for devices %s on hw module %s",
                      supportedDevice->toString().c_str(), hwModule->getName());
                continue;
            }
            for (const auto &device : availProfileDevices) {

                if (!device->isAttached()) {
                    device->attach(hwModule);
                }
            }
            if (mPrimaryOutput == 0 &&
                    outProfile->getFlags() & AUDIO_OUTPUT_FLAG_PRIMARY) {
                mPrimaryOutput = outputDesc;
            }
            addOutput(output, outputDesc);
            setOutputDevices(outputDesc,
                             DeviceVector(supportedDevice),
                             true,
                             0,
                             NULL);
        }
```

先简单说下找supportedDevice 的过程，先找到route中的sources中有我们的outProfile的route，把这些route中的sink中device或到一起就是这个outProfile的supportedDevices，然后判断supportedDevices是否包含defalutedevice，包含则此outProfile的supportedDevice就是defalutedevice，否则从mAvailableOutputDevices（即attachedDevices标签中devices）中找到第一个包含的就是我们的supportedDevice。然后根据我们的outProfile来new SwAudioOutputDescriptor同时调用其open方法，传入我们的supportdevice和一个int类型的output的引用。我们继续看下SwAudioOutputDescriptor的open函数吧

```cpp
status_t SwAudioOutputDescriptor::open(const audio_config_t *config,
                                       const DeviceVector &devices,
                                       audio_stream_type_t stream,
                                       audio_output_flags_t flags,
                                       audio_io_handle_t *output)
{
    mDevices = devices;
    const String8& address = devices.getFirstValidAddress();
    audio_devices_t device = devices.types();

    audio_config_t lConfig;

    if (config == nullptr) {
        lConfig = AUDIO_CONFIG_INITIALIZER;
        lConfig.sample_rate = mSamplingRate;
        lConfig.channel_mask = mChannelMask;
        lConfig.format = mFormat;
    } else {
        lConfig = *config;
    }

    if ((mProfile->getFlags() & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) &&
            lConfig.offload_info.format == AUDIO_FORMAT_DEFAULT) {
        flags = (audio_output_flags_t)(flags | AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD);
        lConfig.offload_info = AUDIO_INFO_INITIALIZER;
        lConfig.offload_info.sample_rate = lConfig.sample_rate;
        lConfig.offload_info.channel_mask = lConfig.channel_mask;
        lConfig.offload_info.format = lConfig.format;
        lConfig.offload_info.stream_type = stream;
        lConfig.offload_info.duration_us = -1;
        lConfig.offload_info.has_video = true;
        lConfig.offload_info.is_streaming = true;
    }

    mFlags = (audio_output_flags_t)(mFlags | flags);

    ALOGV("opening output for device %s profile %p name %s",
          mDevices.toString().c_str(), mProfile.get(), mProfile->getName().string());

    status_t status = mClientInterface->openOutput(mProfile->getModuleHandle(),
                                                   output,
                                                   &lConfig,
                                                   &device,
                                                   address,
                                                   &mLatency,
                                                   mFlags);
    LOG_ALWAYS_FATAL_IF(mDevices.types() != device,
                        "%s openOutput returned device %08x when given device %08x",
                        __FUNCTION__, mDevices.types(), device);

    if (status == NO_ERROR) {
        LOG_ALWAYS_FATAL_IF(*output == AUDIO_IO_HANDLE_NONE,
                            "%s openOutput returned output handle %d for device %08x",
                            __FUNCTION__, *output, device);
        mSamplingRate = lConfig.sample_rate;
        mChannelMask = lConfig.channel_mask;
        mFormat = lConfig.format;
        mId = AudioPort::getNextUniqueId();
        mIoHandle = *output;
        mProfile->curOpenCount++;
    }

    return status;
}
```

这里没啥好说的，又是mClientInterface我们知道又调到了AudioFlinger中，看下AudioFlinger中的源码

```cpp
status_t AudioFlinger::openOutput(audio_module_handle_t module,
                                  audio_io_handle_t *output,
                                  audio_config_t *config,
                                  audio_devices_t *devices,
                                  const String8& address,
                                  uint32_t *latencyMs,
                                  audio_output_flags_t flags)
{
    ALOGI("openOutput() this %p, module %d Device %#x, SamplingRate %d, Format %#08x, "
              "Channels %#x, flags %#x",
              this, module,
              (devices != NULL) ? *devices : 0,
              config->sample_rate,
              config->format,
              config->channel_mask,
              flags);

    if (devices == NULL || *devices == AUDIO_DEVICE_NONE) {
        return BAD_VALUE;
    }

    Mutex::Autolock _l(mLock);

    sp<ThreadBase> thread = openOutput_l(module, output, config, *devices, address, flags);
    if (thread != 0) {
        if ((flags & AUDIO_OUTPUT_FLAG_MMAP_NOIRQ) == 0) {
            PlaybackThread *playbackThread = (PlaybackThread *)thread.get();
            *latencyMs = playbackThread->latency();

            playbackThread->ioConfigChanged(AUDIO_OUTPUT_OPENED);

            if ((mPrimaryHardwareDev == NULL) && (flags & AUDIO_OUTPUT_FLAG_PRIMARY)) {
                ALOGI("Using module %d as the primary audio interface", module);
                mPrimaryHardwareDev = playbackThread->getOutput()->audioHwDev;

                AutoMutex lock(mHardwareLock);
                mHardwareStatus = AUDIO_HW_SET_MODE;
                mPrimaryHardwareDev->hwDevice()->setMode(mMode);
                mHardwareStatus = AUDIO_HW_IDLE;
            }
        } else {
            MmapThread *mmapThread = (MmapThread *)thread.get();
            mmapThread->ioConfigChanged(AUDIO_OUTPUT_OPENED);
        }
        return NO_ERROR;
    }

    return NO_INIT;
}
```

先看下openOutput_l

```cpp
sp<AudioFlinger::ThreadBase> AudioFlinger::openOutput_l(audio_module_handle_t module,
                                                            audio_io_handle_t *output,
                                                            audio_config_t *config,
                                                            audio_devices_t devices,
                                                            const String8& address,
                                                            audio_output_flags_t flags)
{

    AudioHwDevice *outHwDev = findSuitableHwDev_l(module, devices);
    if (outHwDev == NULL) {
        return 0;
    }

    if (*output == AUDIO_IO_HANDLE_NONE) {
        *output = nextUniqueId(AUDIO_UNIQUE_ID_USE_OUTPUT);
    } else {

        ALOGE("openOutput_l requested output handle %d is not AUDIO_IO_HANDLE_NONE", *output);
        return 0;
    }

    mHardwareStatus = AUDIO_HW_OUTPUT_OPEN;

    if (!(flags & (AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD | AUDIO_OUTPUT_FLAG_DIRECT))) {

        if (kEnableExtendedPrecision) {

        }
        if (kEnableExtendedChannels) {

        }
    }

    AudioStreamOut *outputStream = NULL;
    status_t status = outHwDev->openOutputStream(
            &outputStream,
            *output,
            devices,
            flags,
            config,
            address.string());

    mHardwareStatus = AUDIO_HW_IDLE;

    if (status == NO_ERROR) {
        if (flags & AUDIO_OUTPUT_FLAG_MMAP_NOIRQ) {
            sp<MmapPlaybackThread> thread =
                    new MmapPlaybackThread(this, *output, outHwDev, outputStream,
                                          devices, AUDIO_DEVICE_NONE, mSystemReady);
            mMmapThreads.add(*output, thread);
            ALOGV("openOutput_l() created mmap playback thread: ID %d thread %p",
                  *output, thread.get());
            return thread;
        } else {
            sp<PlaybackThread> thread;
            if (flags & AUDIO_OUTPUT_FLAG_COMPRESS_OFFLOAD) {
                thread = new OffloadThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created offload output: ID %d thread %p",
                      *output, thread.get());
            } else if ((flags & AUDIO_OUTPUT_FLAG_DIRECT)
                    || !isValidPcmSinkFormat(config->format)
                    || !isValidPcmSinkChannelMask(config->channel_mask)) {
                thread = new DirectOutputThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created direct output: ID %d thread %p",
                      *output, thread.get());
            } else {
                thread = new MixerThread(this, outputStream, *output, devices, mSystemReady);
                ALOGV("openOutput_l() created mixer output: ID %d thread %p",
                      *output, thread.get());
            }
            mPlaybackThreads.add(*output, thread);
            mPatchPanel.notifyStreamOpened(outHwDev, *output);
            return thread;
        }
    }

    return 0;
}
```

我们先开看下AudioHwDevice的openOutputStream，关键代码如下

```cpp
status_t status = outputStream->open(handle, devices, config, address);
```

又调用到了AudioStreamOut的open中，

```cpp
status = hwDev()->openOutputStream(
                handle,
                devices,
                customFlags,
                &customConfig,
                address,
                &outStream);

```

这里的hwDev（）是DeviceHalInterface而我们在分析openDevice的时候

```cpp
*device = new DeviceHalHidl(result);
```

已经初始化过这个DeviceHalInterface了，因此这里又调到了DeviceHalHidl的openOutputStream中，

```cpp
 mDevice->openOutputStream(
              handle,
              hidlDevice,
              hidlConfig,
              EnumBitfield<AudioOutputFlag>(flags),

```

mDevice是在 new DeviceHalHidl的时候传入的Device，

```cpp
Return<void> Device::openOutputStream(int32_t ioHandle, const DeviceAddress& device,
                                      const AudioConfig& config, AudioOutputFlagBitfield flags,
                                      openOutputStream_cb _hidl_cb) {
    AudioConfig suggestedConfig;
    auto [result, streamOut] =
        openOutputStreamImpl(ioHandle, device, config, flags, &suggestedConfig);
    _hidl_cb(result, streamOut, suggestedConfig);
    return Void();
}
```

又调用了openOutputStreamImpl

```cpp
std::tuple<Result, sp<IStreamOut>> Device::openOutputStreamImpl(int32_t ioHandle,
                                                                const DeviceAddress& device,
                                                                const AudioConfig& config,
                                                                AudioOutputFlagBitfield flags,
                                                                AudioConfig* suggestedConfig) {
    audio_config_t halConfig;
    HidlUtils::audioConfigToHal(config, &halConfig);
    audio_stream_out_t* halStream;
    ALOGV(
        "open_output_stream handle: %d devices: %x flags: %#x "
        "srate: %d format %#x channels %x address %s",
        ioHandle, static_cast<audio_devices_t>(device.device),
        static_cast<audio_output_flags_t>(flags), halConfig.sample_rate, halConfig.format,
        halConfig.channel_mask, deviceAddressToHal(device).c_str());
    int status =
        mDevice->open_output_stream(mDevice, ioHandle, static_cast<audio_devices_t>(device.device),
                                    static_cast<audio_output_flags_t>(flags), &halConfig,
                                    &halStream, deviceAddressToHal(device).c_str());
    ALOGV("open_output_stream status %d stream %p", status, halStream);
    sp<IStreamOut> streamOut;
    if (status == OK) {
        streamOut = new StreamOut(this, halStream);
    }
    HidlUtils::audioConfigFromHal(halConfig, suggestedConfig);
    return {analyzeStatus("open_output_stream", status, {EINVAL} ), streamOut};
}
```

终于看到了一点希望最终调到了hal下的open_output_stream中，最终调用了audio_hw中的adev_open_output_stream打开我们的输出通道，回到我们的AudioFlinger中继续看然后根据我们传入的flag来创建对应的thread，这里基本都是new MixerThread，最后把new的thread加入到mPlaybackThreads.add(*output, thread)中。到此也就基本差不多了。

AudioPolicyManager的initialize的过程我们简单分析了两个函数分别是loadHwModule和openOutput，整个过程从AudioPolicy到到AudioFlinger最终到AudioHal下有些复杂，明天我会将时序图补上供参考。然后再继续分析initialize中的剩余部分。
