# 视频混剪未选择 BGM 时禁止默认配乐技术方案

## 1. 背景

在视频混剪批量生成时，用户未选择 BGM 素材，但最终成片仍出现背景音乐。

排查后发现，前端在部分场景下即使没有选择 BGM，也可能仍携带音量相关字段，例如：

- `backgroundMusicVolume`
- `bgmVolume`
- `backgroundMusicVolumeRatio`
- `bgmVolumeRatio`

如果后端只根据音量字段生成阿里云 `BackgroundMusicConfig`，但没有真实的 `BackgroundMusicArray`，可能导致阿里云智能媒体服务按默认背景音乐逻辑处理。

## 2. 设计原则

本次优先采用后端兜底方案，尽量不改前端。

核心原则：

> 只有用户真实选择了 BGM 音频文件，后端才允许向阿里云提交背景音乐配置。

也就是说：

- 没有 `backgroundMusicArray`：不传 `BackgroundMusicArray`，也不传 `BackgroundMusicConfig`。
- 有 `backgroundMusicArray`：正常传 `BackgroundMusicArray`，并按前端音量配置传 `BackgroundMusicConfig.Volume`。

## 3. 后端改动点

文件：

`ruoyi-system/src/main/java/com/ruoyi/system/service/impl/SubmitBatchMediaProducingServiceImpl.java`

### 3.1 改动前逻辑

后端会直接解析 BGM 音量：

```java
Double backgroundMusicVolume = resolveBackgroundMusicVolume(mediaProducingDto);
if (backgroundMusicVolume != null)
{
    JSONObject backgroundMusicConfig = new JSONObject();
    backgroundMusicConfig.put("Volume", backgroundMusicVolume);
    editingConfig.put("BackgroundMusicConfig", backgroundMusicConfig);
}
```

问题是：即使没有真实 BGM 文件，只要前端传了音量字段，后端也会生成 `BackgroundMusicConfig`。

### 3.2 改动后逻辑

新增是否选择 BGM 的判断：

```java
boolean hasBackgroundMusic = hasBackgroundMusic(mediaProducingDto);
Double backgroundMusicVolume = hasBackgroundMusic ? resolveBackgroundMusicVolume(mediaProducingDto) : null;
if (hasBackgroundMusic && backgroundMusicVolume != null)
{
    JSONObject backgroundMusicConfig = new JSONObject();
    backgroundMusicConfig.put("Volume", backgroundMusicVolume);
    editingConfig.put("BackgroundMusicConfig", backgroundMusicConfig);
}
```

新增辅助方法：

```java
private boolean hasBackgroundMusic(MediaProducingDto mediaProducingDto)
{
    return mediaProducingDto != null && !defaultList(mediaProducingDto.getBackgroundMusicArray()).isEmpty();
}
```

## 4. 出入参变化

### 4.1 前端入参

前端可以保持现状。

即使入参中存在音量字段，只要没有真实 BGM 地址，后端也会忽略 BGM 配置：

```json
{
  "backgroundMusicArray": [],
  "backgroundMusicVolume": 78,
  "bgmVolume": 78,
  "backgroundMusicVolumeRatio": 0.78,
  "bgmVolumeRatio": 0.78
}
```

### 4.2 提交给阿里云的参数

未选择 BGM 时，阿里云请求中不应出现：

```json
{
  "inputConfig": {
    "BackgroundMusicArray": []
  },
  "editingConfig": {
    "BackgroundMusicConfig": {
      "Volume": 0.78
    }
  }
}
```

正确结果应为：

```json
{
  "inputConfig": {},
  "editingConfig": {}
}
```

说明：这里只展示和 BGM 相关的字段，其他混剪配置仍正常保留。

### 4.3 选择 BGM 时

如果用户真实选择了 BGM：

```json
{
  "backgroundMusicArray": [
    "https://zhixiaotian-test.oss-cn-beijing.aliyuncs.com/zhixiaotian/1/bgm.mp3"
  ],
  "backgroundMusicVolume": 78
}
```

后端仍会正常提交：

```json
{
  "inputConfig": {
    "BackgroundMusicArray": [
      "https://zhixiaotian-test.oss-cn-beijing.aliyuncs.com/zhixiaotian/1/bgm.mp3"
    ]
  },
  "editingConfig": {
    "BackgroundMusicConfig": {
      "Volume": 0.78
    }
  }
}
```

## 5. 测试覆盖

文件：

`ruoyi-system/src/test/java/com/ruoyi/system/service/impl/SubmitBatchMediaProducingServiceImplTest.java`

新增两个测试：

### 5.1 未选择 BGM 时忽略音量字段

测试方法：

```java
buildBatchSubmitDebugShouldIgnoreBgmVolumeWhenNoBackgroundMusicSelected()
```

验证点：

- `inputConfig` 不包含 `BackgroundMusicArray`
- `editingConfig` 不包含 `BackgroundMusicConfig`

### 5.2 选择 BGM 时保留音量控制

测试方法：

```java
buildBatchSubmitDebugShouldKeepBgmVolumeWhenBackgroundMusicSelected()
```

验证点：

- `inputConfig` 包含 `BackgroundMusicArray`
- `editingConfig.BackgroundMusicConfig.Volume = 0.78`

## 6. 验证命令

```bash
cd /Users/kexiaobin/Desktop/其他/熵变智元/智小天pc-0419/local-debug/zhixiaotian-houduan-may
mvn -pl ruoyi-system -Dtest=SubmitBatchMediaProducingServiceImplTest#buildBatchSubmitDebugShouldIgnoreBgmVolumeWhenNoBackgroundMusicSelected,SubmitBatchMediaProducingServiceImplTest#buildBatchSubmitDebugShouldKeepBgmVolumeWhenBackgroundMusicSelected test
```

验证结果：

```text
Tests run: 2, Failures: 0, Errors: 0
BUILD SUCCESS
```

## 7. 结论

本方案不要求前端立即调整。

后端通过 `backgroundMusicArray` 判断用户是否真实选择 BGM，避免仅因音量字段存在就生成阿里云背景音乐配置，从而解决“未选择 BGM 但成片出现默认背景音乐”的问题。

修改一下


再修改一下


hit-fix修改 修改
