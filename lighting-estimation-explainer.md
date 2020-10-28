# WebXR Device API - Lighting Estimation
This document explains the portion of the WebXR APIs that enable developers to render augmented reality content that reacts to real world lighting.

## Introduction

"Lighting Estimation" is implemented by AR platforms using a combination of sensors, cameras, algorithms, and machine learning.  Lighting estimation provides input to rendering algorithms and shaders to ensure that the shading, shadows, and reflections of objects appear natural when presented in a diverse range of settings.

The `XRLightProbe` interface exposes the values that the platform offer to WebXR rendering engines. The corresponding retrieval method, `XRSession.requestLightProbe()`, returns a promise and is only accessible once an AR session has started. The promise may be resolved on the same frame or multiple frames later, depending on the platform capabilities. In some cases, the promise may fail, indicating that the lighting values are not available for that session.

Once an `XRLightProbe` has been created it can be used to query an `XRLightEstimate` each frame with the XRFrame.getLightEstimate() method. The light estimate provides both an ambient illumination estimate in the form of spherical harmonics and an estimate of the direction and intensity of the primary light source in the user's environment. The light probe can also be used to query an estimated cube map representing the users environment from the `XRWebGLBinding.getReflectionCubeMap()` method.

Although modern render engines support multiple light and reflection probes in a scene, the WebXR API returns only a single `XRLightProbe`, representing the global approximated lighting values to be used in the area in close proximity to the viewer. When future platforms become capable of reporting multiple probes with precise locations away from the viewer, such support could be implemented additively without breaking changes.

The orientation of the lighting information is reported relative to the `XRLightProbe.probeSpace`, which can be queried each `XRFrame` like any other space, and may be the same as an existing XRReferenceSpace. As it may be computationally expensive to rotate spherical harmonics values and texture cubes, the probeSpace enable the same values to be used in multiple orientations.

It is possible to treat a synthetic VR scene as the environment that AR content will be mixed in to. In this case, the platform will be able to report the lighting estimation using the geometry of the VR scene.  As the WebXR API does not specifically express if the world is synthetic or real, AR content is to be written the same, without such knowledge.  Such "AR in VR" techniques do not affect the WebXR specification directly and are beyond the scope of this text.

## Physically Based Units

The lighting estimation values represent luminance and colors that may be outside the gamut of the output device.  Direct sunlight can project 5000 nits at full power, while a typical display may emit only 250-500 nits.  The objects in a scene will attenuate the power of the sun and reflect a smaller portion towards the viewer.  Even if the display can only represent such a limited gamut (such as SRGB, P3, or Rec 2020), intermediate lighting calculations used by shaders involve scaling up small values and attenuating large values outside of the displayed gamut.  When the lighting calculation results in a color that can not be displayed, the resulting value will be altered by a variety of post processing effects to match the rendering intent and aesthetic chosen by the content authors.

Luminance values are expressed in nits (cd/m^2).  Nits are used by some native platform lighting estimation API's and the media-capabilities API.  User agents will translate the values returned by native platforms to nits for consistency.

As lighting is scene-relative as opposed to display-relative, the luminance values are encoded linearly with no gamma curve.  Most modern render engines perform intermediate calculations in linear space and can accept such values directly.  If an engine performs intermediate calculations in a color space encoded with gamma, such as sRGB, care must be taken when converting the values.  After scaling the values, the result may include components above 1.0 or below 0.0.  Naive implementations that clamp RGB components independently will result in erraneous hue and saturation for out-of-gamut colors.

## Cube Map Textures

HDR Cube Map textures, as returned by `XRWebGLBinding.getReflectionCubeMap()`, provide all the information about light sources and indirect bounces needed to accurately render PBR materials that are diffuse, glossy, and visibly reflective.  Image based lighting effects utilizing such textures are simple to implement and perform well for VR and AR rendering.  Unfortunately, such cube map textures require a lot of video memory and can often represent the environment from a limited range of locations where such a map was captured.

HDR Cube Map textures are commonly used to implement "Reflection Probes" in modern rendering engines.

## Spherical Harmonics

SH (Spherical Harmonics) are used as a more compact alternative to HDR cube maps by storing a small number of coefficient values describing a fourier series over the surface of a sphere.  SH can effectively compress cube maps, while retaining multiple lights and directionality.  Due to their lightweight nature, many spherical harmonics probes can be used within a scene, be interpolated, or be calculated for locations nearer to the lit objects.

WebXR's Lighting Estimation module supports up to 9 spherical harmonics coefficients per RGB color component, for a total of 27 floating point scalar values. This enables the level 2 (3rd) order of details. If a platform can not supply all 9 coefficients, it can pass 0 for the higher order coefficients resulting in an effectively lower frequency reproduction. This may be used to communicate a a simple global "ambient" term when more detailed lighting information is either not available or not allowed by the user.

This "Spherical harmonics probe" format is used by most modern rendering engines, including Unity, Unreal, and Threejs.

## Shadows

When an HDR Cube Map texture is available, shadows only have to consider occlusion of other rendered objects in the scene.

When a HDR Cube Map texture is not available, or the typical soft shadow effects of image based lighting are too costly to implement, the `XRLightEstimate.primaryLightDirection` and `XRLightEstimate.primaryLightIntensity` can be used to render shadows cast by the most prominent light source.

## Security Implications
The lighting estimation API shares many potential privacy risks with the [ambient light sensor API](https://www.w3.org/TR/ambient-light/#security-and-privacy), including:

- profiling: Lighting estimation can leak information about user’s use patterns and surrounding. This information can be used to enhance user profiling and behavioral analysis.
- cross device linking: Two devices can access web sites that include the same third-party script that correlates lighting levels over time.
- cross device communication

Lighting estimation also provides additional opportunities for side channel attacks and fingerprinting risks, discussed in this section.

### Feature Descriptor

In order for the applications to signal their interest in accessing lighting estimation during a session, the session must be requested with appropriate feature descriptor. The string `light-estimation` is introduced by this module as new valid feature descriptor. `light-estimation` enables the light estimation feature, and is required for `XRSession.requestLightProbe()` to resolve.

The inline XR device MUST NOT support the `light-estimation` feature.

### XRLightProbe

The `XRLightProbe` itself contains no lighting values, but is used to retrieve the current lighting state with each `XRFrame`.

```js
let lightProbe = await xrSession.requestLightProbe();
```

The position and orientation in space that the lighting is estimated relative to is communicated with the `probeSpace` attribute, which is an `XRSpace`. The `probeSpace` may update it's pose over time as the user moves around their environment.

```js
let probePose = xrFrame.getPose(lightProbe.probeSpace, xrReferenceSpace);
```

### XRLightEstimate

`XRLightEstimate` returns sufficient information to render objects that appear to fit into their environment, with highly diffuse surfaces or high frequency normal maps which would result in a wide NDF (normal distribution function).  Highly polished objects may be represented with a non-physically based illusion of glossiness with a specular highlight effect sensitive only to the primary light direction.  Reflections will be unable to reproduce detailed images of the environment without a cube map.

The lighting estimation returned by the WebXR API explicitly describes the real world environment in proximity to the user.  By default, only low spatial frequency and low temporal frequency information should be returned by the WebXR API.  Even when a platform can directly produce higher spatial and temporal frequency information, the browser must apply a low pass filter with an aim to mitigate the risk of untrusted content identifying the geolocation of the user or of profiling their environment.

```js
// Using Three.js to demonstrate
let threeDirectionalLight = new THREE.DirectionalLight();
// THREE.LightProbe is Three.js' spherical harmonics-based light type.
let threeLightProbe = new THREE.LightProbe();

let lightProbe = await xrSession.requestLightProbe();

function onXRFrame(t, xrFrame) {
  let lightEstimate = xrFrame.getLightEstimate(lightProbe);

  let intensity = Math.max(1.0,
                  Math.max(lightEstimate.primaryLightIntensity.x,
                  Math.max(lightEstimate.primaryLightIntensity.y,
                           lightEstimate.primaryLightIntensity.z)));

  threeDirectionalLight.position.set(lightEstimate.primaryLightDirection.x,
                                     lightEstimate.primaryLightDirection.y,
                                     lightEstimate.primaryLightDirection.z);
  threeDirectionalLight.color.setRGB(lightEstimate.primaryLightIntensity.x / intensity,
                                     lightEstimate.primaryLightIntensity.y / intensity,
                                     lightEstimate.primaryLightIntensity.z / intensity);
  threeDirectionalLight.intensity = intensity;

  threeLightProbe.sh.fromArray(lightEstimate.sphericalHarmonicsCoefficients);

  // ... other typical frame loop stuff.
}
```

Only first term of the `XRLightEstimate.sphericalHarmonicsCoefficients` is guaranteed to be available either due to user privacy settings or the capabilities of the platform. When a values estimated from the user's environment are not available the `primaryLightDirection` will report `(0.0, 1.0, 0.0, 0.0)`, representing a light shining straight down from above, and the `primaryLightIntensity` will report `(0.0, 0.0, 0.0, 1.0)`, representing no illumination.

Combined with other factors, such as the user's IP address, even the low frequency information returned with XRLightProbe increases the fingerprinting risk.  The XRLightProbe should only be accessible during an active WebXR session.

### Reflection Cube Map

The cube map returned by passing an `XRLightProbe` to `XRWebGLBinding.getReflectionCubeMap()` enables efficient and simple to implement image based lighting. PBR shaders can index the mip map chain of the environment cube to reduce the memory bandwidth required while integrating multiple samples to match wider NDF's.

While the estimated cube map is expected to update over time to better reflect the user's environment as they move around those changes are unlikely to happen with every `XRFrame`. Since creating and processing the cube map is potentially expensive, especially if mip maps are needed, pages can listen to the `reflectionchange` event on the `XRLightProbe` to determine when an updated cube map needs to be retrieved.

```js
let glBinding = new XRWebGLBinding(xrSession, gl);

let lightProbe = await xrSession.requestLightProbe();
let glCubeMap = glBinding.getReflectionCubeMap(lightProbe);

lightProbe.addEventListener('reflectionchange', () => {
  glCubeMap = glBinding.getReflectionCubeMap(lightProbe);
});
```

By default the cube map will be returned as a 8BPP sRGB texture. Some underlying runtimes may deliver the text data in a different "native" format however, such high dynamic range formats. The light probe's preferred internal format is reported by the `XRLightProbe.preferredReflectionCubeMapFormat`, which may alternately be specified when querying the cube map. Querying the cube map using the preferred format ensures the minimal amount of conversion needs to happen, which in turn may be faster and experience less data loss. PAssing any value other than `"srgb8"` or the light probe's `preferredReflectionCubeMapFormat` to `getReflectionCubeMap()` will cause a `null` texture to be returned.

```js
let glCubeMap = glBinding.getReflectionCubeMap(lightProbe, lightProbe.preferredReflectionCubeMapFormat);
```

UA's may provide real-time reflection cube maps, captured by cameras or other sensors reporting high frequency spatial information. To access such real-time cube maps, the `camera` feature policy must be enabled for the origin. `XRWebGLBinding.getReflectionCubeMap()` should only return real-time cube maps following user consent equivalent to requesting access to the camera.

UA's may provide a reflection cube map that was pre-created by the end user, which may differ from the environment while the `XRSession` is active. In particular, the user may choose to manually capture a reflection cube map at an earlier time when sensitive information or people are not present in the environment.

#### Cube Map Open Questions:
  - Is there a size tolerance for the cube maps re: user consent? ARCore's cube maps are limited to 16px per side, which seems difficult to get sensitive information out of.
  - Should/can we dictate the format the cube maps return in. ARCore prefers Half Float values, which is hard for WebGL 1 compatibility and cubemap gen. No idea what format ARKit's textures are in.
  - Should we handle mipmapping for the user. My gut and ARCore says no, but I'm not sure what ARKit does here either.
  - Should we allow for a single texture to be returned multiple times? Seems potentially fragile but may incur unnecessary readback on iOS.

### Temporal and Spatial Filtering

Rapid changes to incoming light can provide information about a user's surroundings that can lead to fingerprinting and side-channel attacks on user privacy.

As an example, a light switch can be flipped in a room, causing the lighting estimation of two users in the same room to simultaneously change. If the precise time of the change can be observed, it can be inferred that the two users are co-located in the same physical space.

Another example occurs when a nearby display is playing a video, such as an advertisement. The light from the display reflects off many surfaces in the room, contributing to the observable ambient light estimate. A timeline of light intensity changes can uniquely identify the video that is playing, even if the monitor is not in direct line-of-sight to the XR device sensors.

A UA MUST apply temporal and spatial filtering of the light estimation to avoid such attacks. A low-pass filter effect can be achieved by averaging the values over the last several seconds. For single scalar values representing light intensity or color, such as `XRLightEstimation.sphericalHarmonicsCoefficients` and `XRLightEstimation.primaryLightIntensity` this can be applied directly with a box-kernel. SH's have a convenient property that they can be summed and interpolated by simply interpolating their coefficients, assuming their orientation is not changing.  These SH coefficients can also be filtered as scalar values with a box-kernel.

Filtered values MUST be first quantized before the box-kernel is applied. Any vectors, such as `XRLightEstimation.primaryLightDirection` should be quantized in 3D space and always return a unit vector.

## Appendix A: Proposed partial IDL
This is a partial IDL and is considered additive to the core IDL found in the main [explainer](explainer.md).

```webidl
partial interface XRSession {
  Promise<XRLightProbe> requestLightProbe();
}

partial interface XRFrame {
  XRLightEstimate? getLightEstimate(XRLightProbe lightProbe);
};

enum XRReflectionCubeMapFormat {
  "srgb8",
  "hdr16f",
};

[SecureContext, Exposed=Window]
partial interface XRLightProbe : EventTarget {
  readonly attribute XRSpace probeSpace;
  readonly attribute XRReflectionCubeMapFormat preferredReflectionCubeMapFormat;
  attribute EventHandler onreflectionchange;
};

[SecureContext, Exposed=Window]
partial interface XRLightEstimate {
  readonly attribute Float32Array sphericalHarmonicsCoefficients;
  readonly attribute DOMPointReadOnly primaryLightDirection;
  readonly attribute DOMPointReadOnly primaryLightIntensity;
};

// See https://github.com/immersive-web/layers for definition.
partial interface XRWebGLBinding {
  WebGLTexture? getReflectionCubeMap(XRLightProbe lightProbe, XRReflectionCubeMapFormat format="srgb8");
};
```
