![Status:Draft](https://img.shields.io/badge/Draft-blue)
## Image Plane Support for USD

## Contents
- [Background](#background)
  - [Interchange](#interchange)
  - [Optical System](#optical-system)
- [Proposal](#proposal)
  - [Why Separate Schemas?](#why-separate-schemas?)
  - [Properties](#properties)
- [Hydra Implementation](#hydra-implementation)
- [Alternative Implementation Strategies](#alternative-implementation-strategies)
- [Open Questions](#open-questions)

## Background

The majority of work in VFX is centered around augmenting existing footage, so it is important to view animation in the context of that footage (it doesn't matter if the animation looks good if it doesn't look integrated). A part of this augmentation process includes challenges like de-keystoning, stretching anamorphic shots, match move solving to replicate the camera's movement in the virtual environment and rotoscoping which is when you would want to mask/matte parts of live action footage for further edits such as drawing over it. Often times this footage is displayed on a 2D card, which we're referring to as a back plate, that is rendered like any other object in the scene. Back plates can also be useful in photogrammetry. One possible workflow as an example: you can load a mesh of the subject and array of cameras (provided by a company like Lidar Lounge) into Maya from which the mesh can then be simplified and exported as USD. Then you can import it all into Mari, so the cameras become projectors; with each having the path to a back plate's image, that can each then be projected onto the mesh. Without back plates, the utility of tools like usdview in matchmove, virtual production, photogrammetry, and other workflows is quite restricted and requires compositing before work can be evaluated in context.

Another feature that also has its utility in the VFX/animation space is being able to portray an image that's at an infinite distance and always in focus irrespective of whether the camera itself is focused at infinity. Instead of projecting an image onto a card in the scene, you can think of this as projecting an image onto the sensor itself pre-render; we are essentially filling the pixel sample values before rendering. We will denote this feature as an image plane. In a potential scene, the image plane would be the far-away castle that rests atop a hill as a backdrop and back plates would hold the vines and branches to "shoot" through.

### Interchange
By adding support for this feature we expand the use of USD when interchanging with other software like [Maya](https://help.autodesk.com/view/MAYAUL/2025/ENU/?guid=GUID-E2490B87-087E-476A-9C1D-A917D009001A), [Blender](https://docs.blender.org/manual/en/latest/modeling/meshes/import_images_as_planes.html), [Renderman](https://rmanwiki-26.pixar.com/space/REN26/19661891/PxrImageDisplayFilter), and [Nuke](https://www.nukepedia.com/tools/gizmos/3d/imageplane3d/) ([another impl](https://www.nukepedia.com/tools/gizmos/3d/imageplane/)). In particular we are aware of interest in using USD with [Composure](https://dev.epicgames.com/documentation/en-us/unreal-engine/composure#plate) (Unreal Engine's compositing plugin) that would require back plate support to do so. Our proposal will cover what capabilities these DCC's back plate equivalents provide and outline the differences compared with our back plates.

### Optical System
Before we delve into the schema we're proposing, we'll give a brief overview of our camera model to better understand how transformations to these planes will affect the final image and compare it to the current model of usdGeom camera.
(Side by side comparision of our thin lens opticcal system versus the pinhole model our system currently uses)
![](opticsSys.png)
Fig 1. back plate/image plane API camera model (left); our system's camera model (right)

Let us begin by defining a few terms:

- Principal plane H: A common simplification in optics is to reduce a complex system into two principal planes (H_near and H_far) that represent the point at which all refraction occurs. These planes can be calculated by taking incident rays parallel to the optical axis and finding their intersection with the emerging rays that intersect the focal point. For practical purposes we are assuming an infinitely thin lens such that H_near and H_far are equal, which lets us work with a single principal plane H. This can also be seen as the plane in which the apex of the viewing frustum lies. Note that this is a mathematical reference, it does not physically exist.
- Intrinsic camera origin: The origin of the components within the camera sits at the camera sensor when it is at the focal length. We did consider using the apex of the frustum as our camera origin; however that location is a bit arbitrary given that this point sits at a virtual reference. Instead we've chosen to use our current model since the focus distance when capturing photos is often measured from this point. In this model, our camera is still considered +Y up looking down the -Z axis.
- Extrinsic camera origin: UsdGeomCamera, GfCamera, and GfFrustum all use a pinhole model such that the apex of the frustum is the camera origin. In order to maintain backwards compatibility, computations under a pinhole model, where the origin of the camera sits at the principal plane H, will need to be offset by the focal length to refactor the intrinsic camera origin to the principal plane. This must take place before any other transformations on the camera such as rotating the camera along the X axis such that it matches the stage's orientation.
- Focus distance: In cinematography this is the distance from the camera sensor to the plane at which objects in the scene will be in focus. Note that in optics this distance is measured from the apex of the frustum.
- Focal length: the distance from the principal plane H to the focal point when the lens is focused at infinity.
- F-stop: the ratio of the focal length to the diameter of the camera's aperture that controls how much light is allowed through the lens of the camera.

## Proposal
This proposal aims to add native support for back plates to USD and will outline the framework for image planes though we will leave the latter for when there is better infrastructure to implement image planes in Hydra. During our discussions we outlined two use cases that our schema could fulfill; one being pre-render processing, where pixels sample values are filled before rendering (image planes), and mid-render, where these planes are shot as any other object in the scene (back plates). For our back plate model, you can think of the camera as a projector and the back plate as a screen trying to capture the projection. However, unlike a projector, the transformations are applied on the back plate itself. You can think of it like tilting the screen to correct for keystoning as opposed to adjusting the lens in the projector. An image plane on the other hand refers to the camera sensor, such that images constrained to the sensor are always in focus and any transformation will directly modify the sensor. For example shifting the image plane back can shorten the focus distance, and increasing the sensor size will also expand the field of view. Key implementation differences between the two are a camera should only have one image plane whereas multiple back plates can be associated with the camera, and back plates should be composed during rendering (meaning additional depth, color, and alpha information will be taken into account during the render), while image planes are composed before. Neither will emit light. With image planes and back plates being two distinct concepts, we'll propose two different schemas for them, single-apply for image planes and multi-apply for back plates. Note that projections are out of scope seeing as they involve shading in addition to depending on usdGeom to provide something to project onto.

### Why Separate Schemas?
If we implement a Multi-Apply API Schema for back plates we can:
- Author attributes directly to the camera
- Author multiple back plates to camera, since multi-apply schemas can be applied several times to the same prim
- multiple back plates could be used to view CG elements in context with rotoscoped foreground elements

If we implement a Single-Apply API Schema for image planes we can:
- Author attributes directly to the camera
- Restrict the user to only authoring one image plane to the camera
- This will need to be explicitly applied since not all camera will have media associated with them.

Other considerations we had:
- Creating a base multi-apply schema that contained all the properties shared among image planes and back plates and could be included as a builtin schema by both
- Creating a shared schema with a property to toggle between the two concepts

These alternative solutions were primarily to reduce the redundancy among the two; however the namespacing of the properties for the former solution would look something like backPlate:imageFitBase:_instance_name_:propertyName for back plates and similarly for image planes, which is not as user friendly. Furthermore we would also need to implement extra logic to ensure that there can only be one image plane authored to each camera. In order to draw a clear distinction between the two concepts as well as simplifying the property namespacing we've opted for two separate schemas.

### Properties
| **Property** | **Back Plate Description** | **Image Plane Description** |
| -------- | ---------------------- | ----------------------- |
| **float width/height**  | Width/height of back plate measured in scene units.  | The width/height of a image plane are measured in 1/10s of a scene unit.  | 
| **float3f scaleXform**   | Scales the projection of the image onto the plate.<sup>1</sup>  | Scales the projection of the image onto the sensor.<sup>1</sup>  | 
| **float3f rotateXform**  | Rotate the plate relative to the camera.<sup>1</sup>  | Rotates the sensor relative to the intrinsic camera origin.<sup>1</sup>  | 
| **float3f translateXform**  | Translates the plate relative to the intrinsic camera origin.<sup>1</sup> | Translates the sensor relative to the intrinsic camera origin.<sup>1</sup>  | 
| **asset image**  | Asset path to the file containing a texture or sequence of textures. Hydra nor Usd textures support extracting images from media; though this is currently a project on the road map. When it is available we will revisit this schema and add media support for image planes as well. The images by default will be centered on the back plate.  | Same as back plate.  | 
| **asset depthChannel**  | File path to the depth channel associated with the image if one exists. If not then the back plate will have uniform depth. The depth channel is a linear, floating point distance value computed as computed_depth = ((value_from_texture - minimum_value) * normalizing_factor) + world_space_offset, where the offsets can be trivially set by the user. The depth channel is not a hardware generated z buffer as these have unknown format and are not generally readable outside of the GPU. | N/A  | 
| **float minDepthOffset**  | Value to shift the depth values if minimum value of current range is not set at 0 if needed. | N/A | 
| **float normalizingFactor**  | Value to scale the texture depth value if needed. | N/A | 
| **float worldSpaceOffset**  | Offset to shift the depth into world space if needed.  | N/A  | 
| **float3f gain**  | Scales the per-channel upper bound of the normalized signal before gamma, analogous to per-channel exposure or slope. Must be >= 0.<sup>2</sup>  | Same as back plate.  | 
| **float3f lift**  | Raises or lowers the signal floor (black level) of the normalized value prior to the gain and gamma stages. Ranges from 0-1.<sup>2</sup>   | Same as back plate. | 
| **float3 gamma**  | Per-channel power applied to the normalized RGB signal after lift and gain.Must be greater than some epsilon value.<sup>2</sup>   | Same as back plate.  | 
| **token visibility ["all","solo","mute"]**  | Toggles the visibility of the back plate to all cameras, only the associated camera, or no cameras, with "solo" being the fallback. Note that in use cases such as in photogrammetry, where you may have many plates in the scene, it can be useful to disable certain back plates to reduce clutter in your workflow.  | N/A  | 
| **bool mute**  | N/A  | Toggles the visibility of the image plane. The fallback value is true.  | 

1. Transformation matrices will be applied in the order of translate•rotate•scale•v,where v is the center/origin of the back plate.
2. Lift, gain, and gamma are applied in the order of (x*(gain-lift)+lift)^(1/gamma).  

Other considerations that will not be included:

- **Measuring width/height as an aspect ratio**: we also considered measuring the dimensions as a ratio determined by the back plate depth in the scene such that the default back plate is "fullscreen"; however this representation was unintuitive given our current model. By defining the intrinsic origin of the camera at the image plane, the depth would need to account for the offset to the frustum apex before computing the final depth. We considered adding an additional depth attribute that could specify the distance of the back plate from the apex of the frustum, and all other transformations would be applied after the shift; however this is a bit redundant given that we already have a translation transformation so to provide a more straightforward set of properties we have decided to use the model presented above.
- **Crop**: describes how the image should be clipped/how thick the border of the image should be. The same effects can be achieved by modifying the transformation of the back plate.
- **Fit**: describes how the image should be adjusted to fit within the dimensions of the card. The same effects can be achieved by modifying the properties of the frustum and transformation of the back plate; however a fit token would be contrary to our model since it describes how an image would be pasted onto a texture card. It does not encapsulate the semantics of a projection that are often needed for digital processing of VFX footage (de-keystoning, anamorphic images etc).
- **Alpha**: If a user wanted to use a texture that included alpha they could provide two source inputs, RGB and opacity, such that Storm can ingest RGB values and the alpha values can be manually wired to an opacity input. USD and Storm, do not understand channel naming so we cannot directly ingest textures with multiple channels. We also opted to not provide an alpha channel attribute in our schema, because the alpha channel is typically not available unless the image was synthetically rendered; for on set applications, RGBD values are captured.
- **UseDepth**: Similar to how HdPrman has a UseAlpha toggle to tell the system whether or not is should look for an alpha channel in it's image file, we could provide a UseDepth bool to tell our system to search for a depth channel at the image file path. Seeing as we already need to trouble the user to wire up alpha separately, we figured it would be more convenient and follow our current pattern and explicitly look for a depth channel.
- **Relation to UsdRenderPass**: While we currently will not be interacting with additional render passes, having individual bools to toggle whether or not image planes are visible and back plates are visible, can be useful for render pass workflows. For instance, a background render pass would want the image plane visible but not the back plates; whereas other render passes may want differently.

#### API Methods
- CreateBackPlate(backPlateName)
- SetBackPlate*Attr(backPlateName, value)
- GetBackPlate*Attr(backPlateName, attribute)
- SetBackPlates([])
- GetBackPlates()

#### A Minimum Usage Example:
```
camera = stage.GetPrimAtPath('/world/cam')
backPlateApi = UsdGeomBackPlateAPI(camera)
backPlateApi.CreateBackPlate("backPlate1")
framesToImages = [(f, fileSequence.frameAt(i) for f in fileSequence.frames()]
imagePlaneApi.SetImagePlaneImageAttr(framesToImages, "backPlate1")
```
##### Which would generate this .usda:
```
def Xform "world" {
    def Camera "cam" (        
        apiSchemas = ["UsdGeomImagePlaneAPI:imagePlane1"]
    ){
        ...
        string[] imagePlanes = ["imagePlane1"]
        asset imagePlane1:image = {
            1001: @/path/to/image.1001.exr@, 
            1002:...
            }
        float imagePlane1:depth = 1000.0
    }
}
```

## DCC Interchange
Here we list the image plane features of [Maya](https://help.autodesk.com/view/MAYAUL/2025/ENU/?guid=GUID-E2490B87-087E-476A-9C1D-A917D009001A), [Blender](https://docs.blender.org/manual/en/latest/modeling/meshes/import_images_as_planes.html), [Composure](https://dev.epicgames.com/documentation/en-us/unreal-engine/composure#plate), [Renderman](https://rmanwiki-26.pixar.com/space/REN26/19661891/PxrImageDisplayFilter), and [Nuke](https://www.nukepedia.com/tools/gizmos/3d/imageplane3d/) ([another impl](https://www.nukepedia.com/tools/gizmos/3d/imageplane/)) and call out the features that we will not support with this proposal. 

| First Header  | Maya | Blender | Nuke |
| ------------- | ------------- | ------------- | ------------- | 
| DCC Features  |  - color grading controls - camera visibility modes  - alpha channels  - luminance  - various modes to fit the image plane onto the camera frame - frame cropping and offsets - non-camera associated image planes  - supports images, animation, and media  |     different types of shading: emission, shadeless, and their BSDF shader
    different render methods: forward vs deferred
    back face culling
    alpha channel support
    various modes to fit the image plane onto the camera frame
    always camera facing toggle
    grouping multiple image planes at once
    supports images and animation  |     choose reference frame + reference camera
    supports images and animation  |
| Interchange with USD  | :warning: Maya supports standard image planes as well as free image planes, which are image planes not associated with a camera. We've observed that the standard use cases for this seem to be a modeling reference, and it is suggested that this specific image plane would be better represented as a texture card.  | Our image planes will not be emissive, thus we will only support Blender's shadeless shader option.  | 

## Engineering Work


### Hydra Integration

- In order to integrate back plates in Hydra, such that it can be rendered in Usdview, we will be defining a filtering scene index to identify back plates in the scene and produce camera associated texture cards for the renderer to process. Scene filters exists as an intermediary step between HdSceneIndices and HdSceneIndexObservers that can accept scene updates and transform/modify those updates to the next filtering scene index or renderer. For our use cases, a part of these transformations can include any color/depth information updates.
- Image planes on the other hand need to be incorporated after rendering. Generating and rearranging Hydra tasks is one option, however it would be complex and is not recommended since the HdTask interface for Hydra 2.0 is still being developed. Instead, one option would be to create a filtering scene index that converts the necessary data to composite the image plane into hydra prims and then the render index will use that data to apply the specified compositions in a render pass. Note that the exact solution is very render specific.

Ideas:
- Use UsdPreviewSurface textures in a way similar to GeomModel texture cards.
- Do the compositing in an Hdx post task

## Risk, Issues, and Caveats
Out of Scope

- **Projection painting**: i.e. enabling users to paint textures onto some geometry is used by ILM for their Mars/Commodore system and also frequently used in gaming as decals. While it would be useful to include this feature, it should be part of the shading system since the painted texture would need to be exported as a material or primvar that the shader node can consume. In our shading pipeline we associate cameras with the projection setup and part of the model build extracts the static xform and stamps it out as a primvar that the shader node can consume.
- **Adding additional render passes**: in order to support image planes, we would need to apply its properties in a post-procesing stage which most likely entails specifying additional render passes in the renderer. It may be possible to create a scene index that wraps the properties that a render index can then detect and apply as a separate render pass; however this method would be very render specific and dependent. The other option is to hold off until more work has been done to improve HdTask's interface in Hydra 2.0 to allow us to generate additional Hydra tasks for an additional render pass.
- **Full compositing graph**: while we do want to support interchange with Composure; this would also likely involve investigating how USD render passes can be used to achieve a similar affect to Composure's layer passes.
- **Displaying media**: USD currently does not support media inputs although when that feature is available we will enable back plates and image planes to display media.

## Use Cases
### Hydra / Usdview
It would be helpful to be able to view CG elements against a backdrop of the production plate when checking exported usd caches.

## Alternative Implementation Strategies
### 1.) As a concrete prim type
Here is a working implementation that was done before Multi-Apply schemas were added to USD.

It's a fully featured solution that is based around Autodesk Maya's "underworld" image plane nodes where many planes 
can live under a camera. This specific implementation is just a quick attempt to move maya image planes across packages 
and the hydra implementation is just a workaround that generates an HdMesh rprim (instead of creating a new image plane 
prim type).

Reference internal PR from Luma: https://github.com/LumaPictures/USD/pull/1

#### Pros:
- Prim attributes like [visibility, purpose] can be leveraged to provide intuitive functionality
- Easy to see Image Planes in a hierarchy
#### Shortfalls:
- Requires relationships between camera + plane prims
- Requires an extra API schema to encode the relationship on the Camera.
- There's the possibility that someone targets something other than an ImagePlane with the relationship, so there are more ways to "get stuff wrong". 

### 2.) Using existing USD types (Plane + material + camera + expressions)
- It is possible to create a UsdPlane with a material and accomplish the same thing as an image plane.
#### Pros:
- No need for this proposal :)
#### Shortfalls:
- Requires users to implement complex authoring code.
- No explicit identity as an "Image Plane". This will make it hard to successfully roundtrip from DCCs to USD.