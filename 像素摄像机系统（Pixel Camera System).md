
----
> **关键词**：Unity、Orthographic Camera、RenderTexture、Upscale、Voxel Grid、Subpixel、Perspective、Pixel Art
---

## 目录

1. [概述](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#%E6%A6%82%E8%BF%B0)
2. [核心思路](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#%E6%A0%B8%E5%BF%83%E6%80%9D%E8%B7%AF)
3. [为什么使用正交相机](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#%E4%B8%BA%E4%BB%80%E4%B9%88%E4%BD%BF%E7%94%A8%E6%AD%A3%E4%BA%A4%E7%9B%B8%E6%9C%BA)
4. [透视相机的难点与可行性](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#%E9%80%8F%E8%A7%86%E7%9B%B8%E6%9C%BA%E7%9A%84%E9%9A%BE%E7%82%B9%E4%B8%8E%E5%8F%AF%E8%A1%8C%E6%80%A7)
5. [完整源码](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#%E5%AE%8C%E6%95%B4%E6%BA%90%E7%A0%81)
    - [1. PixelPerfectURP/RenderTextureFunctions.cs](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#1-pixelperfecturprendertexturefunctionscs)
    - [2. PixelCamera/CanvasViewCamera.cs](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#2-pixelcameracanvasviewcameracs)
    - [3. PixelCamera/UpscaledCanvas.cs](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#3-pixelcameraupscaledcanvascs)
    - [4. PixelCamera/PixelCameraManager.cs (含Tooltips)](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#4-pixelcamerapixelcameramanagercs-%E5%90%ABtooltips)
6. [总结](https://chatgpt.com/c/67afe926-1ed8-8000-8cd3-762da841ec7f?model=o1#%E6%80%BB%E7%BB%93)

---

## 概述

**像素摄像机系统（Pixel Camera System）** 是一套在 Unity 中实现「低分辨率渲染 + 放大显示」的方案，常用于 2D 或 3D 像素风格游戏。

- **低分辨率渲染**：将游戏画面渲染到较小的 `RenderTexture`，从而得到粗糙的“像素化”视觉。
- **放大显示**：再将该低分辨率纹理贴到一个可放大的画布（`UpscaledCanvas`）上，以适配高分辨率屏幕。

与此同时，可借助一些特殊处理（如体素网格移动、亚像素校正）来减少抖动、保证像素边缘的稳定性。

---

## 核心思路

1. **正交相机渲染**  
    在正交模式（Orthographic）下，以一个较低的分辨率（`GameResolution`）去渲染场景。
2. **RenderTexture 缩放**  
    将这个低分辨率 `RenderTexture` 通过一个网格或 UI 画布（`UpscaledCanvas`）进行放大显示。
3. **体素网格 (Voxel Grid) & 亚像素 (Subpixel)**
    - **体素网格移动**：将相机或物体位置对齐到像素网格，避免物体在屏幕上出现亚像素抖动。
    - **亚像素校正**：在画布层面进行细微移动或偏移，以在运动时保持画面流畅，不出现过度跳动。

这套系统包含 **4** 大部分功能：

- **`RenderTextureFunctions`**：用于根据宽高比和不同同步模式生成合适的 `RenderTexture`。
- **`CanvasViewCamera`**：一个专门用于放大画布的相机，正交模式，并且实现了亚像素移动等辅助功能。
- **`UpscaledCanvas`**：实际承载低分辨率纹理的画布对象，可根据需求动态调整大小。
- **`PixelCameraManager`**：系统核心脚本，负责分辨率管理、相机缩放、对齐到像素网格及整体调度。

---

## 为什么使用正交相机

4. **线性对应关系**  
    正交投影下，相机的视口与屏幕像素是线性对应的。也就是说，`orthographicSize` 与像素大小的换算几乎是“一步到位”，更易实现“像素网格对齐”。
5. **无透视变形**  
    正交相机没有远小近大的透视效果，不同深度的物体投影到屏幕的比例一致，方便像素级别的运算及舍入。
6. **更易控制抖动**  
    在正交模式下，对象每移动一个像素单位，就能与屏幕像素一一对应，不会出现因相机视角或距离变化带来的抖动问题。

---

## 透视相机的难点与可行性

### 难点

7. **像素大小不恒定**  
    透视相机下，同一对象在不同距离或角度时，屏幕像素并不对应相同的世界单位大小，难以进行统一的“网格对齐”。
8. **失去简单的亚像素修正**  
    透视使得物体可能在画面中产生非线性移动，即使做子像素修正，也只能局限在屏幕空间处理，效果不如正交相机直观。
9. **投影矩阵更复杂**  
    若要在世界坐标层面对齐像素，需要考虑自定义投影矩阵或 Shader 量化（quantization），实现成本高。

### 可行的思路

- **依旧采用“低分辨率渲染 + 放大”**：  
    用透视相机把场景渲染到低分辨率 `RenderTexture` 后，直接贴到画布上进行放大，可以获得像素化外观，但“对齐”会较难完美。
- **局部或后处理**：
    - 对特定对象（如主角）进行特殊处理，将它们以正交或特殊对齐方案单独渲染再合成。
    - 使用后处理（Post-processing）Shader 做 pixelation、color depth 降低等效果。

> 如果要求“真正的像素格无抖动”，在透视场景中会更复杂，且难以像正交相机一样实现完美对齐。但是如果仅仅是想“看起来像素风”，则低分辨率+Upscale 依旧是一种可行方案。

---

## 完整源码

以下示例代码皆基于 **Unity C#**，可直接复制到相应脚本文件中使用。为方便说明，做了中文注释。

> **提示**：示例中部分类命名空间不同 (`PixelPerfectURP` vs `PixelCamera`)，请根据实际项目需要进行修改或统一。

---

### 1. PixelPerfectURP/RenderTextureFunctions.cs

```csharp
using UnityEngine;

namespace PixelPerfectURP
{
    /// <summary>
    /// 用于指定渲染分辨率同步模式的枚举
    /// </summary>
    public enum ResolutionSynchronizationMode
    {
        /// <summary>根据给定高度自动计算宽度</summary>
        SetHeight,
        /// <summary>根据给定宽度自动计算高度</summary>
        SetWidth,
        /// <summary>使用给定的宽度和高度</summary>
        SetBoth
    }

    public static class RenderTextureFunctions
    {
        /// <summary>
        /// 根据宽高比和同步模式，返回最终的纹理分辨率
        /// </summary>
        /// <param name="aspect">宽高比 (width / height)</param>
        /// <param name="resolution">原始分辨率</param>
        /// <param name="resolutionSynchronizationMode">同步模式</param>
        /// <returns>计算后的纹理分辨率</returns>
        public static Vector2Int TextureResultion(
            float aspect,
            Vector2Int resolution,
            ResolutionSynchronizationMode resolutionSynchronizationMode)
        {
            switch (resolutionSynchronizationMode)
            {
                case ResolutionSynchronizationMode.SetHeight:
                    return new Vector2Int(
                        Mathf.RoundToInt(resolution.y * aspect),
                        resolution.y);

                case ResolutionSynchronizationMode.SetWidth:
                    return new Vector2Int(
                        resolution.x,
                        Mathf.RoundToInt(resolution.x / aspect));

                case ResolutionSynchronizationMode.SetBoth:
                    return new Vector2Int(resolution.x, resolution.y);

                default:
                    return Vector2Int.one;
            }
        }

        /// <summary>
        /// 创建一个指定分辨率的 RenderTexture，并返回
        /// </summary>
        /// <param name="textureSize">纹理尺寸 (width, height)</param>
        /// <returns>创建成功的 RenderTexture；若失败返回 null</returns>
        public static RenderTexture CreateRenderTexture(Vector2Int textureSize)
        {
            var newTexture = new RenderTexture(
                textureSize.x,
                textureSize.y,
                32,
                RenderTextureFormat.ARGB32)
            {
                filterMode = FilterMode.Point // 像素化风格：点采样
            };

            if (newTexture.Create())
            {
                return newTexture;
            }
            else
            {
                return null;
            }
        }
    }
}
```

---

### 2. PixelCamera/CanvasViewCamera.cs

```csharp
namespace PixelPerfectURP
{
    using UnityEngine;

    /// <summary>
    /// 用于在画布上平滑显示像素相机结果的相机类
    /// </summary>
    [ExecuteInEditMode]
    public class CanvasViewCamera : MonoBehaviour
    {
        /// <summary>画布所使用的相机组件</summary>
        private Camera canvasCamera;

        /// <summary>
        /// 相机宽高比
        /// </summary>
        public float Aspect => this.canvasCamera.aspect;

        /// <summary>
        /// 视图相机的缩放倍数
        /// </summary>
        public float Zoom { get ; private set; }

        void OnEnable()
        {
            this.Zoom = -1;
            this.Initialize();
        }

        /// <summary>
        /// 初始化：确保相机为正交模式
        /// </summary>
        void Initialize()
        {
            if (!this.TryGetComponent(out this.canvasCamera))
            {
                Debug.LogError("A camera component is required on CanvasViewCamera object!");
            }
            else if (this.canvasCamera.orthographic == false)
            {
                Debug.LogWarning("Pixel camera system works best in orthographic mode. Switching to orthographic!");
                this.canvasCamera.orthographic = true;
            }
        }

        /// <summary>
        /// 调整视图相机在画布上的位置，做亚像素修正
        /// </summary>
        /// <param name="targetViewPosition">目标物体在 Viewport 中的位置 [0..1]</param>
        /// <param name="canvasLocalScale">画布的局部缩放</param>
        public void AdjustSubPixelPosition(Vector2 targetViewPosition, Vector2 canvasLocalScale)
        {
            // 将 (0.5,0.5) 视为中心点，计算偏移位置
            var localPosition = (targetViewPosition - new Vector2(0.5f, 0.5f)) * canvasLocalScale;
            this.transform.localPosition = new Vector3(localPosition.x, localPosition.y, -1f);
        }

        /// <summary>
        /// 设置视图相机的缩放
        /// </summary>
        /// <param name="inputZoom">输入的缩放倍数</param>
        /// <param name="halfCanvasHeight">画布高度的一半，用于换算正交尺寸</param>
        public void SetZoom(float inputZoom, float halfCanvasHeight)
        {
            this.canvasCamera.orthographicSize = inputZoom * halfCanvasHeight;
            this.Zoom = inputZoom;
        }

        /// <summary>
        /// 设置视图相机的近远裁剪平面
        /// </summary>
        public void SetClipPlanes(float near, float far)
        {
            this.canvasCamera.nearClipPlane = near;
            this.canvasCamera.farClipPlane = far;
        }
    }
}
```

---

### 3. PixelCamera/UpscaledCanvas.cs

```csharp
namespace PixelPerfectURP
{
    using UnityEngine;

    /// <summary>
    /// 用于放大显示游戏相机渲染结果的画布
    /// </summary>
    [ExecuteInEditMode]
    public class UpscaledCanvas : MonoBehaviour
    {
        /// <summary>材质中存储低分辨率纹理的属性名</summary>
        const string materialVariableName = "_LowResTexture";

        /// <summary>画布所使用的材质引用</summary>
        Material canvasMaterial;

        void OnEnable()
        {
            this.Initialize();
        }

        /// <summary>
        /// 初始化放大画布：检查渲染器、材质以及父物体的缩放
        /// </summary>
        void Initialize()
        {
            if (!this.TryGetComponent<MeshRenderer>(out var meshRenderer))
            {
                Debug.LogError("MeshRenderer is required on UpscaledCanvas!");
            }
            else
            {
                this.canvasMaterial = meshRenderer.sharedMaterial;
                if (this.canvasMaterial == null)
                {
                    Debug.LogError("canvasMaterial is null. Please set a valid material on MeshRenderer!");
                }
            }

            if (this.transform.parent != null && this.transform.parent.localScale != Vector3.one)
            {
                Debug.LogWarning("UpscaledCanvas's parent localScale should be Vector3.one. Auto-correcting!");
                this.transform.parent.localScale = Vector3.one;
            }
        }

        /// <summary>
        /// 判断材质中是否包含 RenderTexture 属性
        /// </summary>
        public bool MaterialHasRenderTexture => this.canvasMaterial.HasProperty(materialVariableName);

        /// <summary>
        /// 根据给定宽高比和目标高度重置画布大小
        /// </summary>
        /// <param name="aspect">宽高比</param>
        /// <param name="canvasHeight">目标高度</param>
        public void ResizeCanvas(float aspect, float canvasHeight)
        {
            this.transform.localScale = new Vector3(canvasHeight * aspect, canvasHeight, 1f);
        }

        /// <summary>
        /// 将指定的 RenderTexture 设置到材质属性中
        /// </summary>
        /// <param name="renderTexture">低分辨率渲染结果</param>
        public void SetCanvasRenderTexture(RenderTexture renderTexture)
        {
            this.canvasMaterial.SetTexture(materialVariableName, renderTexture);
        }
    }
}
```

---

### 4. PixelCamera/PixelCameraManager.cs (含Tooltips)

```csharp
namespace PixelCamera
{
    using UnityEngine;

    /// <summary>
    /// Tooltips for Unity inspector fields
    /// </summary>
    public static class Tooltips
    {
        public const string TT_FOLLOWED_TRANSFORM = "Transform that this camera follow for pixel perfect corrections.";
        public const string TT_GRID_MOVEMENT = "Camera moves on a voxel grid to avoid jittering for stationary objects.";
        public const string TT_SUB_PIXEL = "Subpixel adjustments to counter the blocky movement when snapping to a grid.";
        public const string TT_FOLLOW_ROTATION = "Should the camera also follow the rotation of the target transform?";
        public const string TT_GAME_RESOLUTION = "The resolution of the game's render texture. Lower values look more pixelated.";
        public const string TT_RESOLUTION_SYNCHRONIZATION_MODE = "How 'GameResolution' should be calculated or synchronized with the display aspect.";
        public const string TT_CONTROL_GAME_ZOOM = "Should this script control the game's orthographic size?";
        public const string TT_GAME_ZOOM = "Controls how zoomed in the scene is by adjusting the camera's orthographic size.";
        public const string TT_VIEW_ZOOM = "Zoom level for the view camera, preserving pixel size while changing how big the final view is.";
    }

    /// <summary>
    /// 管理整个像素摄像机系统的脚本
    /// </summary>
    [ExecuteInEditMode]
    public class PixelCameraManager : MonoBehaviour
    {
        [Tooltip(Tooltips.TT_FOLLOWED_TRANSFORM)]
        public Transform FollowedTransform;

        [Header("Settings")]
        [Tooltip(Tooltips.TT_GRID_MOVEMENT)]
        public bool VoxelGridMovement = true;
        [Tooltip(Tooltips.TT_SUB_PIXEL)]
        public bool SubpixelAdjustments = true;
        [Tooltip(Tooltips.TT_FOLLOW_ROTATION)]
        public bool FollowRotation = true;

        [Header("Resolution")]
        [Tooltip(Tooltips.TT_RESOLUTION_SYNCHRONIZATION_MODE)]
        public ResolutionSynchronizationMode resolutionSynchronizationMode = ResolutionSynchronizationMode.SetHeight;

        [Tooltip(Tooltips.TT_GAME_RESOLUTION)]
        public Vector2Int GameResolution = new Vector2Int(640, 360);

        [Header("Zoom")]
        [Tooltip(Tooltips.TT_CONTROL_GAME_ZOOM)]
        public bool ControlGameZoom = true;
        [Tooltip(Tooltips.TT_GAME_ZOOM)]
        public float GameCameraZoom = 5f;
        [Tooltip(Tooltips.TT_VIEW_ZOOM)]
        [Range(-1f, 1f)]
        public float ViewCameraZoom = 1f;

        Camera gameCamera;
        CanvasViewCamera viewCamera;
        UpscaledCanvas upscaledCanvas;

        float renderTextureAspect;

        void OnEnable()
        {
            this.Initialize();
        }

        void LateUpdate()
        {
            this.UpdateCameraSystem();
        }

        /// <summary>
        /// 相机像素在世界空间中对应的大小
        /// </summary>
        float PixelWorldSize
            => 2f * this.gameCamera.orthographicSize / this.gameCamera.pixelHeight;

        /// <summary>
        /// 获取当前相机的目标纹理分辨率
        /// </summary>
        Vector2Int TargetTextureResolution
            => this.gameCamera.targetTexture == null
               ? Vector2Int.left
               : new Vector2Int(
                     this.gameCamera.targetTexture.width,
                     this.gameCamera.targetTexture.height);

        /// <summary>
        /// 将给定世界坐标对齐到像素网格上
        /// </summary>
        public Vector3 PositionToGrid(Vector3 worldPosition)
        {
            // 将世界坐标转到相机本地方向
            var localPosition = this.transform.InverseTransformDirection(worldPosition);
            // 换算成像素尺寸
            var localPositionInPixels = localPosition / this.PixelWorldSize;
            // 四舍五入到整像素
            var integerMovement = (Vector3)Vector3Int.RoundToInt(localPositionInPixels);
            // 再转换回世界坐标
            var movement = integerMovement * this.PixelWorldSize;
            return (movement.x * this.transform.right)
                 + (movement.y * this.transform.up)
                 + (movement.z * this.transform.forward);
        }

        /// <summary>
        /// 安全设置游戏相机正交大小，避免出现 size = 0 的情况
        /// </summary>
        float SetGameZoom(float zoom)
        {
            var checkedZoom = Mathf.Approximately(zoom, 0f) ? 0.01f : zoom;
            this.gameCamera.orthographicSize = checkedZoom;
            return checkedZoom;
        }

        /// <summary>
        /// 同步视图相机的裁剪平面
        /// </summary>
        void SynchronizeClipPlanes()
        {
            this.viewCamera.SetClipPlanes(
                0f,
                this.gameCamera.farClipPlane - this.viewCamera.transform.localPosition.z);
        }

        /// <summary>
        /// 初始化，确保获取相机、视图相机和画布引用，并进行层级检查
        /// </summary>
        private void Initialize()
        {
            // 获取游戏相机
            if (this.gameCamera == null)
            {
                if (!this.TryGetComponent(out this.gameCamera))
                {
                    Debug.LogError("Camera component not found on PixelCameraManager!");
                }
            }

            // 获取视图相机
            if (this.viewCamera == null)
            {
                this.viewCamera = FindAnyObjectByType(typeof(CanvasViewCamera)) as CanvasViewCamera;
                if (this.viewCamera == null)
                {
                    Debug.LogError("viewCamera is null. Please assign a CanvasViewCamera in the scene!");
                }
            }

            // 获取放大画布
            if (this.upscaledCanvas == null)
            {
                this.upscaledCanvas = FindAnyObjectByType(typeof(UpscaledCanvas)) as UpscaledCanvas;
                if (this.upscaledCanvas == null)
                {
                    Debug.LogError("upscaledCanvas is null. Please assign an UpscaledCanvas in the scene!");
                }
            }

            // 检查层级关系
            if (this.transform.parent == null)
            {
                Debug.LogError("No parent object found. Check your prefab setup!");
            }
            if (this.transform.parent != null && this.transform.parent.childCount > 2)
            {
                Debug.LogWarning("PixelCameraManager's parent usually only has 2 children: This manager and the target transform.");
            }

            // 被跟随的变换
            if (this.FollowedTransform == null)
            {
                Debug.LogError("FollowedTransform is null. Please set it in the inspector.");
            }

            // 初始化时先同步裁剪平面
            this.SynchronizeClipPlanes();
        }

        /// <summary>
        /// 设置新的渲染纹理到画布和游戏相机，并记录当前宽高比
        /// </summary>
        void SetRenderTexture(float aspect, RenderTexture newRenderTexture)
        {
            this.upscaledCanvas.SetCanvasRenderTexture(newRenderTexture);
            this.gameCamera.targetTexture = newRenderTexture;
            this.renderTextureAspect = aspect;
        }

        /// <summary>
        /// 每帧更新整个像素相机系统
        /// </summary>
        void UpdateCameraSystem()
        {
            // 1. 检测宽高比或分辨率是否变化
            var aspectRatioChanged = this.renderTextureAspect != this.viewCamera.Aspect;
            var pixelResolutionChanged = this.GameResolution != this.TargetTextureResolution;
            var resizeCanvas = false;

            if (aspectRatioChanged || pixelResolutionChanged || this.gameCamera.targetTexture == null)
            {
                // 根据视图相机实际宽高比进行同步计算
                this.GameResolution = RenderTextureFunctions.TextureResultion(
                    this.viewCamera.Aspect,
                    this.GameResolution,
                    this.resolutionSynchronizationMode);

                // 释放旧纹理
                if (this.gameCamera.targetTexture != null)
                {
                    this.gameCamera.targetTexture.Release();
                }

                // 创建新纹理
                var newRenderTexture = RenderTextureFunctions.CreateRenderTexture(this.GameResolution);

                // 设置到画布和相机
                this.SetRenderTexture(this.viewCamera.Aspect, newRenderTexture);
                resizeCanvas = true;
            }
            else if (Application.isEditor && this.upscaledCanvas.MaterialHasRenderTexture)
            {
                // 在编辑器下，如已有正确的 RenderTexture，仍重设以防变化
                this.SetRenderTexture(this.renderTextureAspect, this.gameCamera.targetTexture);
                resizeCanvas = true;
            }

            // 2. 处理游戏相机的缩放
            var orthographicSizeChanged = this.gameCamera.orthographicSize != this.GameCameraZoom;

            if (!this.ControlGameZoom)
            {
                // 若不由本脚本控制，则同步实际相机尺寸到 GameCameraZoom
                this.GameCameraZoom = this.gameCamera.orthographicSize;
                resizeCanvas = true;
            }

            if (this.ControlGameZoom && orthographicSizeChanged)
            {
                // 若由本脚本控制且尺寸变了，则更新相机正交尺寸
                this.GameCameraZoom = this.SetGameZoom(this.GameCameraZoom);
                resizeCanvas = true;
            }

            // 3. 处理视图相机的缩放
            if (orthographicSizeChanged || pixelResolutionChanged || this.ViewCameraZoom != this.viewCamera.Zoom)
            {
                // 防止分辨率极小导致越界
                var canvasOnScreenLimit = 1 - (2f / this.GameResolution.y);
                if (this.GameResolution.y < 3)
                {
                    canvasOnScreenLimit = 1f;
                    Debug.LogWarning("GameResolution is too small, unexpected behavior may occur.");
                }

                // 保证 ViewCameraZoom 不为 0 且在 -1~1 之间
                this.ViewCameraZoom = Mathf.Approximately(this.ViewCameraZoom, 0f)
                    ? 0.01f
                    : Mathf.Clamp(this.ViewCameraZoom, -1f, 1f);

                // 设置视图相机缩放
                this.viewCamera.SetZoom(
                    this.ViewCameraZoom * canvasOnScreenLimit,
                    this.GameCameraZoom);
            }

            // 4. 如需，重新调整画布大小
            if (resizeCanvas)
            {
                var gameResolutionAspect = (float)this.GameResolution.x / this.GameResolution.y;
                this.upscaledCanvas.ResizeCanvas(gameResolutionAspect, this.GameCameraZoom * 2f);
            }

            // 5. 更新相机 Transform
            // 跟随目标旋转
            if (this.FollowRotation)
            {
                this.transform.rotation = this.FollowedTransform.rotation;
            }

            // 跟随目标位置
            if (this.VoxelGridMovement)
            {
                this.transform.position = this.PositionToGrid(this.FollowedTransform.position);
            }
            else
            {
                this.transform.position = this.FollowedTransform.position;
            }

            // 6. 亚像素校正
            if (this.SubpixelAdjustments)
            {
                var targetViewPosition = this.gameCamera.WorldToViewportPoint(this.FollowedTransform.position);
                this.viewCamera.AdjustSubPixelPosition(
                    targetViewPosition,
                    this.upscaledCanvas.transform.localScale);
            }
        }
    }
}
```

---

## 总结

10. **正交相机的优势**：
    - 线性映射、没有透视失真，适合像素化对齐。
11. **流程**：
    - 使用 `Game Camera`（正交）渲染到低分辨率纹理
    - 用 `UpscaledCanvas` 把该纹理放大到屏幕上显示
    - `CanvasViewCamera` 可以实现额外的平滑缩放、亚像素移动
    - `PixelCameraManager` 统一管理分辨率、对齐和更新逻辑。
12. **可扩展到透视相机**：
    - 可继续用低分辨率 `RenderTexture` + 放大方式，但 **像素网格移动** 在透视下需要更复杂的处理，而且效果不一定如正交相机理想。
    - 对抖动敏感的部分需单独量化或用后处理（Post-processing）Shader 模拟像素风。

通过本方案，Unity 中的像素风格游戏可以在高分辨率屏幕上保持低分辨率的独特视觉，并拥有可控的缩放和抖动消除能力。希望对您有所帮助，祝开发顺利！

---
