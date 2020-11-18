# Security and Privacy Questionnaire

This document answers the [W3C Security and Privacy
Questionnaire](https://www.w3.org/TR/security-privacy-questionnaire/) for the
WebXR Lighting Estimation specification. Note that the Lighting Estimation feature is only exposed during an active WebXR Immersive AR session, and that many risks are shared with the [ambient light sensor API](https://www.w3.org/TR/ambient-light/#security-and-privacy)

**What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

This feature builds an understanding of the light sources in the user’s environment, as well as generating a low-resolution texture representing the various shapes that might be reflected in a shiny object placed at a specific spot in the scene.

Sites would use this information in order to make a rendered object appear to fit more naturally into the user’s environment.

**Is this specification exposing the minimum amount of information necessary to power the feature?**

The specification allows exposing a texture representing the reflection map, and a set of spherical harmonics for the ambient light, as well as the color and direction of the dominant light. A series of mitigations, including temporal and spatial filtering and quantization, are [recommended](https://github.com/immersive-web/lighting-estimation/blob/main/lighting-estimation-explainer.md#temporal-and-spatial-filtering)

**How does this specification deal with personal information or personally-identifiable information or information derived thereof?**

There are no direct PII exposed by this specification. The mapping of the user's environment and the knowledge that two users may be in the same space are the only derived information that could be done via this API. Other WebXr features (notably hit-test) already potentially expose information about the user’s environment, and the potential to identify two users as being in the same space can be mitigated with the temporal filtering mentioned above.

**How does this specification deal with sensitive information?**

The specification allows a user agent to restrict the quality of the lighting information provided a page, as well as the resolution of the texture for the reflection map. In addition, the specification encourages UAs to not update these values in real-time, this helps to mitigate the potential to determine that two users are in the same physical location.

The lighting estimation feature must be requested before starting a WebXR session, which allows the user agent to show a prompt specifically requesting user approval.

It’s worth noting that the WebXR Hit Test and Depth Modules already expose more environment information than the lighting estimation can do.

**Does this specification introduce new state for an origin that persists across browsing sessions?**

No.

**What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?**

None.

**Does this specification allow an origin access to sensors on a user’s device**

No. However, in order to return lighting estimation data, the platform may use various sensors. The origin never has direct access to these sensors as part of the specification.

**What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.**

This specification isn't directly exposing any data to the origin but can be used to get information about the user's physical environment.

**Does this specification enable new script execution/loading mechanisms?**

No.

**Does this specification allow an origin to access other devices?**

No.

**Does this specification allow an origin some measure of control over a user agent’s native UI?**

No.

**What temporary identifiers might this this specification create or expose to the web?**

None.

**How does this specification distinguish between behavior in first-party and third-party contexts?**

It is an extension to WebXR which is by default blocked for third-party contexts and can be controlled via a Feature Policy flag.

**How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?**

The specification does not mandate a different behavior.

**Does this specification have a "Security Considerations" and "Privacy Considerations" section?**

Yes, there is a a section in the spec. Additionally, the [explainer](https://github.com/immersive-web/lighting-estimation/blob/main/lighting-estimation-explainer.md#security-implications) goes into further detail.

**Does this specification allow downgrading default security characteristics?**

No.

**What should this questionnaire have asked?**

N/a.
