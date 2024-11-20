# Custom Screen Buffers

This is a Godot 4.x project showcasing how to pass custom screen buffers around using viewports. These screen buffers might include things such as colors, depth values, normals... whatever you need for your project! These buffers can then be used in post-processing shaders instead of the built-in `hint_screen_texture`, `hint_depth_texture`, and `hint_normal_roughness_texture`.

## Motivation

As of the creation of this project, Godot does not have a good system for creating and using custom buffers. You can access the color, depth, and normal-roughness buffers in screen-reading shaders (see [this doc](https://docs.godotengine.org/en/stable/tutorials/shaders/screen-reading_shaders.html)). However, these buffers are captured before the transparent geometry pass, so they can never contain any data about transparent objects. In addition, there is no way to pass custom data—only color, depth, and normal can be used. This project shows how to use viewports to overcome both of these limitations.

> **Note:** Currently Godot has a [Rendering Compositor](https://github.com/godotengine/godot-proposals/issues/7916) in the works. One of the plans for the compositor is the ability to capture and pass custom buffers from several points in the rendering pipeline, including after the transparent pass. As of Godot 4.3, there is already a new `CompositorEffect` object which partially implements this design. You can read more about how to use the `CompositorEffect` in [this doc](https://docs.godotengine.org/en/stable/tutorials/rendering/compositor.html).

## Overview

This project has two main parts that are necessary for achieving the custom buffer functionality: a camera scene and a set of object shaders. To see these in action, take a look at either **modular_test_scene.tscn** or **monolithic_test_scene.tscn**. These scenes are almost identical except for a difference in how the object shaders are written. (This difference is explained below.)

### Camera Scene

The scene **main_camera.tscn** is contains the following nodes:
* 4 cameras (including the root `MainCamera`)
* 3 subviewports
* 1 full-screen quad (on which the post-processing shader **screen_shader.gdshader** is applied)

Think of this scene as single camera which captures all the necessary information and passes it to the post-processing effect on the full-screen quad. It captures color on layer 2, depth info on layer 3, and normal info on layer 4. Layer 1 is used as the "main" layer on which the post-processing effect is displayed.

> **Note:** Layer 1 is used in this way so that the color, depth info, or normal info of any rendered object does not interfere with the final result that is displayed by the full-screen quad. Some effects don't require this separation, so the color layer and main layer can be combined. Do whatever makes sense for your project.

You can change the effect simply by editing the `fragment()` function of **screen_shader.gdshader**. In fact, there are already some lines of code that you can uncomment to see the different buffers (color is displayed by default):

```glsl
// Set the screen shader to show info about this pixel (uncomment a line to view)
ALBEDO = pixel_info.color.rgb;                  // Color
//ALBEDO = vec3(pixel_info.depth);                // Depth
//ALBEDO = 0.5 * (pixel_info.normal + vec3(1.0)); // Normal
//ALBEDO = fract(pixel_info.position);            // 3D Position
```

### Object Shaders

I've provided a set of shaders to use on all 3D objects. This allows them to render color, depth info, and normal info on the appropriate layers. I've taken two different approaches to this, which you can select (or customize) based on what makes the most sense for your project:

1. **Modular:** The modular approach uses separate shaders for rendering color, depth info, and normal info. The different shaders are applied to a single mesh by chaining shaders via the `next_pass` property. To see an example of this, take a look at the mesh materials in **modular_test_scene.tscn**.

2. **Monolithic:** The monolithic approach uses a single shader that renders color, depth info, and normal info. To see an example of this, take a look at the mesh materials in **monolithic_test_scene.tscn**.

Both the modular and monolithic approaches separate transparent materials from opaque materials. This is due to the limitations when working with transparent objects, such as sorting (see [3D Rendering Limitations: Transparency Sorting](https://docs.godotengine.org/en/4.3/tutorials/3d/3d_rendering_limitations.html#transparency-sorting). I've separated transparent materials from opaque materials so that objects can be transparent when they need to be, but other objects won't be negatively affected. Obviously you can customize this as needed for your project.

## How To Use

This project can be used either as is (if all you need is the color, depth, and normal buffers), or it can be used as a point of reference as you set up your own project.

### Use As Is

To use this project without any modifications, simply add the **main_camera.tscn** scene wherever you would normally put a camera. Then, modify **screen_shader.gdshader** in order to define your post-processing shader. You will have access to the color, depth, and normal buffers in this shader. Finally, change the materials on all of your 3D objects, choosing one of the following approaches:

* Modular approach: If the object is purely opaque, use the shader **opaque_color_shader.gdshader**. Otherwise, use **transparent_color_shader.gdshader**. After that, make sure to also add **depth_shader.gdshader** and **normal_shader.gdshader** to your objects by chaining them on the `next_pass` property of your materials.

* Monolithic approach: If the object is purely opaque, use the shader **opaque_shader.gdshader**. Otherwise, use **transparent_shader.gdshader**.

Also make sure that these objects are rendering on visual layers 1 through 4.

> **Note:** There may be some limitations to how you can use **main_camera.tscn**. For example, I don't believe it will work properly to have two of them loaded at once in your project.

### Set Up From Scratch

If you want to set this up from scratch in your own project, the general steps to take are as follows:
1. You'll need multiple cameras—one for each buffer that you want to capture. You'll want all of the cameras to have the same properties *except* that they should be on different visual layers. You'll also need to make it so that their 3D transforms match.
2. Add a `SubViewport` for each of the cameras. The viewports will need to be connected to their corresponding cameras. You can do this by writing a little script, which can be put on the cameras or on the viewports. You'll make use of the method `RenderingServer.viewport_attach_camera()` to attach the cameras to the viewports (see [here](https://docs.godotengine.org/en/stable/classes/class_renderingserver.html#class-renderingserver-method-viewport-attach-camera)).
3. Pass the resulting textures from these viewports as `sampler2D` uniforms to your post-processing shader. See [this doc](https://docs.godotengine.org/en/stable/tutorials/shaders/using_viewport_as_texture.html) for an example of how to do this.
4. Finally, you'll need to write a custom shader (or set of shaders) for *all* the objects that you want to be correctly visible in these buffers that you just created. In this object shader, you'll need to output different data based on the layer that is currently being rendered. The current layer(s) are available in the shader built-in `CAMERA_VISIBLE_LAYERS`. (I believe `CAMERA_VISIBLE_LAYERS` is only available in the `vertex()` and `fragment()` functions by default, but you can also pass it as a `varying` to the `light()` function.) In addition, make sure that your objects are rendering on all the correct visual layers.

#### Notes On Passing Pure Data

For any buffer where you plan to pass pure data (unaffected by lights or other visual effects), there are a few things you have to set up to ensure the data is passed unaltered:
* For the camera, add an `Environment` resource with default settings. This ensures that the camera uses linear tonemapping.
* For the viewport, make sure to enable the `use_hdr_2d` property. (The `Environment` resource could also be added to the viewport instead of the camera.)
* For the object shader, if you have one monolithic shader, pass the data through the `EMISSION` output in the `fragment()` function. Then, in the `light()` function, make sure that both `DIFFUSE_LIGHT` and `SPECULAR_LIGHT` are zeroed. If you are using modular shaders, then you can use `render_mode unshaded;` and just write to `ALBEDO` in the `fragment()` function.
