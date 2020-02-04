# WebXR Device API - Lighting Estimation
This document explains the portion of the WebXR APIs that enable developers to render augmented reality content that reacts to real world lighting.

## Introduction

"Lighting Estimation" is implemented by AR platforms using a combination of sensors, cameras, algorithms, and machine learning.  Lighting estimation provides input to rendering algorithms and shaders to ensure that the shading, shadows, and reflections of objects appear natural when presented in a diverse range of settings.

The XRLightProbe and XRReflectionProbe interfaces expose the values that the platform offer to WebXR rendering engines.  Their corresponding accessor functions, XRFrame.getGlobalLightEstimate() and XRFrame.getGlobalReflectionProbe() are only accessible once the first frame of an AR session has started.  The promises may be resolved on the same frame or multiple frames later, depending on the platform capabilities.  In some cases, the promises may fail, indicating that the lighting values are not available at this time and should be requested again at a later time.

Although modern render engines support multiple "Light Probes" and "Reflection Probes" in a scene, the WebXR API returns only a single corresponding XRLightProbe and XRReflectionProbe, representing the global approximated lighting values to be used in the area in close proximity to the viewer.  When future platforms become capable of reporting multiple probes with precise locations away from the viewer, such support could be implemented additively without breaking changes.

The orientation of the lighting information is relative to the XRViewerPose for the XRFrame that getGlobalLightEstimate() or getGlobalReflectionProbe() was requested on.  As it may be computatinonaly expensive to rotate SH and texture cubes, XRLightProbe.sphericalHarmonicsCoefficients() and XRReflectionProbe.orientation() enable the same SH and texture cubes to be used in multiple orientations.

It is possible to treat a synthetic VR scene as the environment that AR content will be mixed in to.  In this case, the platform will be able to report the lighting estimation using the geometry of the VR scene.  As the WebXR API does not specifically express if the world is synthetic or real, AR content is to be written the same, without such knowledge.  Such "AR in VR" techniques do not affect the WebXR specification directly and are beyond the scope of this text.

## Physically Based Units

The lighting estimation values represent luminance and colors that may be outside the gamut of the output device.  Direct sunlight can project 5000 nits at full power, while a typical display may emit only 250-500 nits.  The objects in a scene will attenuate the power of the sun and reflect a smaller portion towards the viewer.  Even if the display can only represent such a limited gamut (such as SRGB, P3, or Rec 2020), intermediate lighting calculations used by shaders involve scaling up small values and attenuating large values outside of the displayed gamut.  When the lighting calculation results in a color that can not be displayed, the resulting value will be altered by a variety of post processing effects to match the rendering intent and aesthetic chosen by the content authors.

Luminance values are expressed in nits (cd/m^2).  Nits are used by some native platform lighting estimation API's and the media-capabilities API.  User agents will translate the values returned by native platforms to nits for consistency.

As lighting is scene-relative as opposed to display-relative, the luminance values are encoded linearly with no gamma curve.  Most modern render engines perform intermediate calculations in linear space and can accept such values directly.  If an engine performs intermediate calculations in a color space encoded with gamma, such as SRGB, care must be taken when converting the values.  After scaling the values, the result may include components above 1.0 or below 0.0.  Naive implementations that clamp RGB components independently will result in erraneous hue and saturation for out-of-gamut colors.

## Global Illumination

Rendering algorithms take into consideration not only the light received by a surface from the light source but also light that has bounced around the scene multiple times before reaching the eye.

Traditional real-time engines had a simple global "ambient" constant value that is added to the real-time shading result.  Engines using such a simple technique can use XRLightProbe.indirectIrradiance, scaled to return the desired effect.  It may also be necessary to apply a gamma curve if the shading is done in SRGB space.

Global illumination describes the collective techniques used to more accurately estimate the light received from indirect reflections.

## Cube Map Textures

HDR Cube Map textures, as created by the XRReflectionProbe provide all the information about light sources and indirect bounces needed to accurately render PBR materials that are diffuse, glossy, and visibly reflective.  Image based lighting effects utilizing such textures are simple to implement and perform well for VR and AR rendering.  Unfortunately, such cube map textures require a lot of video memory and can often represent the environment from a limited range of locations where such a map was captured.

HDR Cube Map textures are commonly used to implement "Reflection Probes" in modern rendering engines.

## Spherical Harmonics

SH (Spherical Harmonics) are used as a more compact alternative to HDR cube maps by storing a small number of coefficient values describing a fourier series over the surface of a sphere.  SH can effectively compress cube maps, while retaining multiple lights and directionality.  Due to their lightweight nature, many SH probes can be used within a scene, be interpolated, or be calculated for locations nearer to the lit objects.

WebXR API supports up to 9 SH coefficients per RGB color component, for a total of 27 floating point scalar values.  This enables the level 2 (3rd) order of details.  If a platform can not supply all 9 coefficients, it can pass 0 for the higher order coefficients resulting in an effectively lower frequency reproduction.  

This "SH probe" format is used by most modern rendering engines, including Unity, Unreal, and Threejs.

## Shadows

When an HDR Cube Map texture is available, shadows only have to consider occlusion of other rendered objects in the scene.

When a HDR Cube Map texture is not available, or the typical soft shadow effects of image based lighting are too costly to implement, the XRLightProbe.primaryLightDirection and XRLightProbe.primaryLightIntensity can be used to render shadows cast by the most prominent light source.

## Security Implications

### Feature Descriptor

In order for the applications to signal their interest in accessing lighting estimation during a session, the session must be requested with appropriate feature descriptor.  The strings `xr-global-light-estimation` and `xr-global-reflection` are introduced by this module as new valid feature descriptors.

`xr-global-light-estimation` enables the global light estimation feature, and is required for promises returned by `getGlobalLightEstimate` called on an `XRFrame` to succeed.

`xr-global-reflection` enables the global reflection feature, and is required for promises returned by `getGlobalReflectionProbe` called on an `XRFrame` to succeed.

The inline XR device MUST NOT be treated as capable of supporting the global light estimation and global reflection features.

UA's may provide a reflection cube map that was pre-created by the end user in response to `xr-global-reflection`, which may differ from the environment while the `XRSession` is active. In particular, the user may choose to manually capture a reflection cube map at an earlier time when sensitive information or people are not present in the environment.

UA's may provide real-time reflection cube maps, captured by cameras or other sensors reporting high frequency spatial information.  To access such real-time cube maps, the `camera` feature policy must also be enabled for the origin.

### XRLightProbe

Only XRLightProbe.indirectIrradiance is guaranteed to be available either due to user privacy settings or the capabilities of the platform.

XRLightProbe returns sufficient information to render objects that appear to fit into their environment, with highly diffuse surfaces or high frequency normal maps which would result in a wide NDF (normal distribution function).  Highly polished objects may be represented with a non-physically based illusion of glossiness with a specular highlight effect sensitive only to the primary light direction.  Reflections will be unable to reproduce detailed images of the environment without an XRReflectionProbe.

The lighting estimation returned by the WebXR API explicitly describes the real world environment in proximity to the user.  By default, only low spatial frequency and low temporal frequency information should be returned by the WebXR API.  Even when a platform can directly produce higher spatial and temporal frequency information, the browser must apply a low pass filter with an aim to mitigate the risk of untrusted content identifying the geolocation of the user or of profiling their environment.

Combined with other factors, such as the user's IP address, even the low frequency information returned with XRLightProbe increases the fingerprinting risk.  The XRLightProbe should only be accessible during an active WebXR session.

### XRReflectionProbe

XRReflectionProbe should only be accessible with a permissions prompt equivalent to requesting access to the camera.  XRReflectionProbe enables efficient and simple to implement image based lighting.  PBR shaders can index the mip map chain of the environment cube to reduce the memory bandwidth required while integrating multiple samples to match wider NDF's.

### Temporal and Spatial Filtering

Rapid changes to incoming light can provide information about a user's surroundings that can lead to fingerprinting and side-channel attacks on user privacy.

As an example, a light switch can be flipped in a room, causing the lighting estimation of two users in the same room to simultaneously change. If the precise time of the change can be observed, it can be inferred that the two users are co-located in the same physical space.

Another example occurs when a nearby display is playing a video, such as an advertisement. The light from the display reflects off many surfaces in the room, contributing to the observable ambient light estimate. A timeline of light intensity changes can uniquely identify the video that is playing, even if the monitor is not in direct line-of-sight to the XR device sensors.

A UA MUST apply temporal and spatial filtering of the light estimation to avoid such attacks. A low-pass filter effect can be achieved by averaging the values over the last several seconds. For single scalar values representing light intensity or color, such as `XRLightProbe.indirectIrradiance` and `XRLightProbe.primaryLightIntensity` this can be applied directly with a box-kernel. SH's have a convenient property that they can be summed and interpolated by simply interpolating their coefficients, assuming their orientation is not changing.  These SH coefficients can also be filtered as scalar values with a box-kernel.

Filtered values MUST be first quantized before the box-kernel is applied.  Any vectors, such as `XRLightProbe.primaryLightDirection` should be quantized in 3d space and always return a unit vector.  Quaternion values, such as `XRReflectionProbe.orientation` and `XRReflectionProbe.orientation` must be maintained in normalized form after quantization.

## Appendix A: Proposed partial IDL
This is a partial IDL and is considered additive to the core IDL found in the main [explainer](explainer.md).

```webidl
partial interface XRFrame {
  Promise<XRLightProbe> getGlobalLightEstimate();
  Promise<XRReflectionProbe> getGlobalReflectionProbe();
};

[SecureContext, Exposed=Window]
partial interface XRLightProbe {
  readonly attribute Float32Array indirectIrradiance;
  readonly attribute Float32Array? primaryLightDirection;
  readonly attribute Float32Array? primaryLightIntensity;
  readonly attribute Float32Array? sphericalHarmonicsCoefficients;
  [SameObject] readonly attribute DOMPointReadOnly? sphericalHarmonicsOrientation;
};

[SecureContext, Exposed=Window]
partial interface XRReflectionProbe {
  [SameObject] readonly attribute DOMPointReadOnly orientation;
  WebGLTexture? createWebGLEnvironmentCube();
};
```