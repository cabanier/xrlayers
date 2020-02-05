
## IDL

```webidl
// general layer configuration
enum XRLayerImageLayout {
  "mono",
  "stereo-side-by-side", // for shared texture and video
  "stereo-top-to-bottom",// for shared texture and video
  "stereo"               // for texture arrays. Invalid for video
};

dictionary XRLayerInit {
  boolean blendTextureSourceAlpha = false;
  boolean chromaticAberrationCorrection = false;
  // pixel layout for stereo layers that are not texture arrays
  XRLayerImageLayout layout = "mono";
  // ignored for mono layers
  XRPose pose;
};

dictionary XRCylinderLayerInit : XRLayerInit {
  float radius;
  float centralAngle;
  float aspectRatio;
};

dictionary XRQuadLayerInit : XRLayerInit {
  float width;
  float height;
};

// configuration for texture layers
 enum XRLayerSampleCountPresets {
  "none",    // no AA, same as 0 or 1
  "default",  // default sample count for MSAA for this platform. e.g. 4
  "min",     // minimum possible sample count, e.g. 2
  "max"      // maximum possible sample count, e.g. 8
};

enum XRLayerStyle {
  "array-of-textures",
  "texture-array",
  "framebuffer"
};

enum XRLayerFixedFoveationLevel {
  "default", // inherit from the session
  "none",
  "low",
  "medium",
  "high"
}

dictionary XRGLLayerInit {
  // projection layers can pass in 0 to get automatic sizing
  unsigned long pixelWidth = 0;
  unsigned long pixelHeight = 0;
  XRLayerStyle style = "array-of-textures";
  XRLayerFixedFoveationLevel foveation = "default"; // only used for equirect
  XRLayerSampleCountPresets sampleCount = "default";
  unsigned long mipCount = 1;
  boolean depth = true; // for framebuffer
  boolean stencil = false; // for framebuffer
  boolean alpha = true; // for framebuffer
};

// implementation
interface XRLayer {
  // Attributes. Is this needed?
  readonly attribute XRLayerInit state; // returns the dictionary with current values.
  void setPose(XRPose pose); // change position of the layer
}

interface mixin XRGLLayer {
  attribute float scaleFactor;
  readonly attribute unsigned long width;
  readonly attribute unsigned long height;
  readonly attribute unsigned long arraySize;
  readonly attribute unsigned long mips;
  // accessing texture marks the layers as dirty? or add method/attribute?
  attribute bool isDirty;
};

interface XRTextureLayer: XRLayer {
  WebGLTexture getTexture(unsigned long offset = 0); // returns texture, texture array.
}
XRTextureLayer includes XRGLLayer;

interface XRFramebufferLayer: XRLayer {
  readonly attribute WebGLFramebuffer framebuffer;
  readonly attribute boolean depth;
  readonly attribute boolean stencil;
  readonly attribute boolean alpha;
};
XRFramebufferLayer includes XRGLLayer;

// are cylinder, etc layer classes needed?
// maybe to change the initial values?

dictionary XRRenderLayerStateInit {
  sequence<XRLayer> layers;
  XRWebGLRenderingContext context;
};

[SecureContext, Exposed=Window] partial interface XRRenderState {
  readonly attribute Array<XRLayer> layers;
  readonly attribute XRWebGLRenderingContext context; // all texture layers have the same context.
};

[SecureContext, Exposed=Window] partial interface XRSession {
  void updateRenderLayerState(optional XRRenderLayerStateInit state = {});

  [NewObject] <XRLayer> requestProjectionLayer(XRLayerInit layer = {}, (HTMLVideoElement or XRGLLayerInit) content = {});
  [NewObject] <XRLayer> requestCylinderLayer(XRCylinderLayerInit layer = {}, (HTMLVideoElement or XRGLLayerInit) content = {});
  [NewObject] <XRLayer> requestQuadLayer(XRQuadLayerInit layer = {}, (HTMLVideoElement or XRGLLayerInit) content = {})
};
```

### The following code creates an immersive-vr XRSession with support for layers.
```javascript
let xrSession;

// TBD opt into layer types?
navigator.xr.requestSession("immersive-vr", {
    requiredFeatures: ['layers']}).then((session) => {
  xrSession = session;
});
</pre>
</div>
```

### Set up the initial layer state with no layers in the scene.
```javascript
let glCanvas = document.createElement("canvas");
let gl = glCanvas.getContext("webgl", { xrCompatible: true });
xrSession.updateRenderState({
  context: gl,
  layers: []});
```

### Set up the initial layer state with a single mono quad layer in the scene.
```javascript
xrSession.requestAnimationFrame((time, xrFrame) => {
  ...
  let glCanvas = document.createElement("canvas");
  let gl = glCanvas.getContext("webgl", { xrCompatible: true });
  let quad_layer = await xrSession.requestQuadLayer({
    width: 1,
    height: 1,
    pose: xrFrame.getViewerPose(xrReferenceSpace)},
    {pixelWidth: 200, pixelHeight: 200});
  xrSession.updateRenderState({
    context: gl,
    layers: [ quad_layer ] });
  ...
}
```

### Set up the initial layer state with a cylinder layer with a stereo video.
```javascript
let video = document.createElement('video');
video.loop = true;
video.src = 'sample.webm';
video.play();

let glCanvas = document.createElement("canvas");
let gl = glCanvas.getContext("webgl", { xrCompatible: true });
let cylinder_layer = await xrSession.requestCylinderLayer({
  layout: "stereo-side-by-side",
  radius: .4,
  centralAngle: Math.PI / 2,
  aspectRatio: 2,
  pose: xrFrame.getViewerPose(xrReferenceSpace)},
  video);
xrSession.updateRenderState({
  context: gl,
  layers: [ cylinder_layer ] });
```

### Set up the initial layer state with a quad and a projection layer and fill them with a random color.
```javascript
function fill_texture(gl, texture) {
  let fb = gl.createFramebuffer();
  gl.bindFramebuffer(gl.FRAMEBUFFER, fb);
  gl.framebufferTexture2D(gl.FRAMEBUFFER,
    gl.COLOR_ATTACHMENT0,
    gl.TEXTURE_2D,
    texture,
    0);
  let time = Date.now();
  gl.clearColor(Math.cos(time / 2000),
    Math.cos(time / 4000),
    Math.cos(time / 6000));
  gl.clear(gl.COLOR_BUFFER_BIT);
  gl.deleteFramebuffer(fb);
}

let glCanvas = document.createElement("canvas");
let gl = glCanvas.getContext("webgl", { xrCompatible: true });
let cylinder_layer = await xrSession.requestCylinderLayer({
  radius: .4,
  centralAngle: Math.PI / 2,
  aspectRatio: 2,
  pose: xrFrame.getViewerPose(xrReferenceSpace)});
let quad_layer = await xrSession.requestQuadLayer({
  width: 1,
  height: 1,
  pose: xrFrame.getViewerPose(xrReferenceSpace)});
xrSession.updateRenderState({
  context: gl,
  layers: [ cylinder_layer, quad_layer ] });
fill_texture(gl, cylinder_layer.getTexture());
fill_texture(gl, quad_layer.getTexture());
```
