# WebXR Device API - Lighting Estimation
This document explains the portion of the WebXR APIs that enable developers to render augmented reality content that reacts to real world lighting.

## Introduction
"Lighting Estimation" is implemented by AR platforms using a combination of sensors, cameras, algorithms, and machine learning.  Lighting estimation provides input to rendering algorithms and shaders to ensure that the shading, shadows, and reflections of objects appears natural when presented in a diverse range of settings.

It is possible to treat a synthetic VR scene as the environment that AR content will be mixed in to.  In this case, the platform will be able to report the lighting estimation using the geometry of the VR scene.  As the WebXR API does not specifically express if the world is synthetic or real, AR content is to be written the same, without such knowledge.  Such "AR in VR" techniques do not affect the WebXR specification directly and are beyond the scope of this text.

## Physically Based Units
The lighting estimation values represent luminance and colors that may be outside the gamut of the output device.  Direct sunlight can project 5000 nits at full power, while a typical display may emit only 250-500 nits.  The objects in a scene will attenuate the power of the sun and reflect a smaller portion towards the viewer.  Even if the display can only represent such a limited gamut (such as SRGB, P3, or Rec 2020), intermediate lighting calculations used by shaders involve scaling up small values and attenuating large values outside of the displayed gamut.  When the lighting calculation results in a color that can not be displayed, the resulting value will be altered by a variety of post processing effects to match the rendering intent and aesthetic chosen by the content authors.

Luminance values are expressed in nits (cd/m^2).  Nits are used by some native platform lighting estimation API's and the media-capabilities API.  User agents will translate the values returned by native platforms to nits for consistency.

As lighting is scene-relative as opposed to display-relative, the luminance values are encoded linearly with no gamma curve.  Most modern render engines perform intermediate calculations in linear space and can accept such values directly.  If an engine performs intermediate calculations in a color space encoded with gamma, such as SRGB, care must be taken when converting the values.  After scaling the values, the result may include components above 1.0 or below 0.0.  Naive implementations that clamp RGB components independently will result in erraneous hue and saturation for out-of-gamut colors.

## Global Illumination

Direct lighting...  Specular...

Indirect lighting...  Diffuse...

## Image based lighting

HDR Cube Maps have infinite direct and indirect lights

SH can compress cube maps, while retaining multiple lights and directionality

Simple ambient light value as fallback

## Shadows

Can create shadows by occluding image based lighting

Simple engines will use a single directional light

## Spherical Harmonics

## Probes

### Single Global Probe

Global probes approximate environment near the viewer

Can expand later to support arbitrarily placed probes

### XRLightProbe

### XRReflectionProbe

User choice may result in promise rejection or degradation.  PBR can use mip-chain.


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