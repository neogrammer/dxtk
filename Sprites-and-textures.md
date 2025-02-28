|[[Getting Started]]|
|---|

In this lesson, we will cover the basics of creating a texture from a bitmap file, and then rendering it using a 2D sprite with various drawing options.

# Setup
First create a new project using the instructions from the previous lessons: [[Using DeviceResources]] and
[[Adding the DirectX Tool Kit]] which we will use for this lesson.

# Background

A [sprite](https://wikipedia.org/wiki/Sprite_(computer_graphics)) is a bitmap rendered at some location on the screen. For Direct3D, this requires making use of:

* A *2D texture resource* which is a surface containing the bitmap image pixel data.

* The *shader resource view* which describes the properties of the texture resource.

> In older versions of Direct3D, the texture resource and shader resource view (SRV) were combined into a single interface. In Direct3D 11, these are distinct objects to allow for different interpretations of the same resource data (such as an array of 2D textures or a cubemap).

* A *sampler state object* which describes how the GPU should handle various aspects of reading the texture data for rendering (such as image filtering and/or tiling of the bitmap).

* A *blend state object* which indicates how the GPU should combine existing information in the render target with the texture data.

* Additional Direct3D objects are also required (*vertex buffer*, *index buffer*, *rasterizer state object*, *input layout*, *vertex shader*, *pixel shader*, and *constant buffer*), but in this tutorial these are handled by [[SpriteBatch]].

# Loading a texture
Start by saving [cat.png](https://github.com/Microsoft/DirectXTK/wiki/images/cat.png) into your new project's directory, and then from the top menu select **Project** / **Add Existing Item...**. Select "cat.png" and click "OK".

In the **Game.h** file, add the following variable to the bottom of the Game class's private declarations:

```cpp
Microsoft::WRL::ComPtr<ID3D11ShaderResourceView> m_texture;
```

In **Game.cpp**, add to the TODO of **CreateDeviceDependentResources**:

```cpp
DX::ThrowIfFailed(
    CreateWICTextureFromFile(device, L"cat.png", nullptr,
    m_texture.ReleaseAndGetAddressOf()));
```

In **Game.cpp**, add to the TODO of **OnDeviceLost**:

```cpp
m_texture.Reset();
```

Build and run the application which will still not be displaying anything but the cornflower blue window, but will have a texture loaded.

<details><summary><i>Click here for troubleshooting advice</i></summary>
<p>If you get a runtime exception, then you may have the "cat.png" in the wrong folder, have modified the "Working Directory" in the "Debugging" configuration settings, or otherwise changed the expected paths at runtime of the application. You should set a break-point on <code>CreateWICTextureFromFile</code> and step into the code to find the exact problem.</p></details>

# Drawing a sprite

In the **Game.h** file, add the following variables to the bottom of the Game class's private declarations:

```cpp
std::unique_ptr<DirectX::SpriteBatch> m_spriteBatch;
DirectX::SimpleMath::Vector2 m_screenPos;
DirectX::SimpleMath::Vector2 m_origin;
```

For the original load of our sprite, we only need a ``ID3D11ResourceShaderView`` object as that's all you require to render. This time, however, we also want to obtain the pixel size of the image which is done by requesting the reader return the ``ID3D11Resource`` interface as well as the SRV.

In **Game.cpp**, modify TODO of **CreateDeviceDependentResources** to be:

```cpp
auto context = m_deviceResources->GetD3DDeviceContext();
m_spriteBatch = std::make_unique<SpriteBatch>(context);

ComPtr<ID3D11Resource> resource;
DX::ThrowIfFailed(
    CreateWICTextureFromFile(device, L"cat.png",
    resource.GetAddressOf(),
    m_texture.ReleaseAndGetAddressOf()));

ComPtr<ID3D11Texture2D> cat;
DX::ThrowIfFailed(resource.As(&cat));

CD3D11_TEXTURE2D_DESC catDesc;
cat->GetDesc(&catDesc);

m_origin.x = float(catDesc.Width / 2);
m_origin.y = float(catDesc.Height / 2);
```

In **Game.cpp**, add to the TODO of **CreateWindowSizeDependentResources**:

```cpp
auto size = m_deviceResources->GetOutputSize();
m_screenPos.x = float(size.right) / 2.f;
m_screenPos.y = float(size.bottom) / 2.f;
```

> If using the <abbr title="Universal Windows Platform">UWP</abbr> template, you also need to add ``m_spriteBatch->SetRotation(m_deviceResources->GetRotation());`` to handle display orientation changes.

> If using Xbox One DirectX 11.X "fast semantics", you also need to add ``m_spriteBatch->SetViewport(m_deviceResources->GetScreenViewport());``.

In **Game.cpp**, add to the TODO of **OnDeviceLost**:

```cpp
m_spriteBatch.reset();
```

In **Game.cpp**, add to the TODO of **Render**:

```cpp
m_spriteBatch->Begin();

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::White, 0.f, m_origin);

m_spriteBatch->End();
```

Build and run, and you should get the following screen:

![Screenshot of cat sprite](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotSpriteCat.PNG)

## Alpha mode
One thing you should notice is that the edges of the cat look strange with a bit of white outline. The problem here is that the ``cat.png`` file's alpha channel is _straight_  alpha (i.e. the pixels are of the form ``(R,G,B,A)``). The default behavior of SpriteBatch, however, is to assume you are using _premultiplied_ alpha (i.e. the pixels are of the form ``(R*A, G*A, B*A, A)``). There are many reasons why using premultiplied alpha is superior, but for now we can fix this mismatch by changing our use of SpriteBatch to use straight alpha blending instead by supplying our own ``ID3D11BlendState`` object. We'll make use of the [[CommonStates]] factory to provide one of the built-in blend state objects.

In the **Game.h** file, add the following variable to the bottom of the Game class's private declarations:

```cpp
std::unique_ptr<DirectX::CommonStates> m_states;
```

In **Game.cpp**, add to the TODO of **CreateDeviceDependentResources**:

```cpp
m_states = std::make_unique<CommonStates>(device);
```

In **Game.cpp**, add to the TODO of **OnDeviceLost**:

```cpp
m_states.reset();
```

In **Game.cpp**, modify the TODO of **Render**:

```cpp
m_spriteBatch->Begin(SpriteSortMode_Deferred, m_states->NonPremultiplied());

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::White, 0.f, m_origin);

m_spriteBatch->End();
```

Build and run again, and you'll get a nice clean cat:

![Screenshot of cat sprite](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotSpriteCat2.PNG)

# Using DDS files for textures

Rather than use a ``PNG`` and the Windows Imaging Component (WIC) to load the texture, a more efficient thing for us to do is to make use of a ``DDS`` file instead. A ``DDS`` file is a container for all kinds of Direct3D resources including 1D and 2D textures, _cubemaps_, _volume maps_, arrays of 1D or 2D textures or cubemaps each optionally with _mipmaps_. It can contain a wide-array of pixel formats and hardware-supported 'block-compression' schemes to save on video memory usage at runtime.

Visual Studio has a built-in system for converting images to DDS as part of the build process, which you can read about [here](https://docs.microsoft.com/visualstudio/designers/using-3-d-assets-in-your-game-or-app).

For this tutorial, we will instead make of use of the [DirectXTex](http://go.microsoft.com/fwlink/?LinkId=248926) **texconv** command-line tool.

1. Download the [Texconv.exe](https://github.com/Microsoft/DirectXTex/releases/latest/download/texconv.exe) from the _DirectXTex_ site save the EXE into your project's folder.
1. Open a Command Prompt and then change to your project's folder.

Then run the following command-line:

    texconv cat.png -pmalpha -m 1 -f BC3_UNORM

Then from the top menu in Visual Studio select **Project** / **Add Existing Item...**. Select [cat.dds](https://github.com/Microsoft/DirectXTK/wiki/media/cat.dds) and click "OK".

Now will return to **Game.cpp** in the **CreateDeviceDependentResources** and change our use of ``CreateWICTextureFromFile`` to ``CreateDDSTextureFromFile``:

```cpp
DX::ThrowIfFailed(
    CreateDDSTextureFromFile(device, L"cat.dds",
        resource.GetAddressOf(),
    m_texture.ReleaseAndGetAddressOf()));
```

Note that since we used the option ``-pmalpha``, we should also make sure we change back to the default ``Begin`` in our **Render** because our "cat.dds" has premultiplied alpha in it.

In **Game.cpp**, modify the TODO of **Render**:

```cpp
m_spriteBatch->Begin();

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::White, 0.f, m_origin);

m_spriteBatch->End();
```

Build and run we are rendering our 'clean' cat with premultiplied alpha:

![Screenshot of cat sprite](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotSpriteCat2.PNG)

## Technical notes

* The switch ``-pmalpha`` causes the **texconv** command-line tool to convert the image to premultiplied alpha before saving the ``.dds`` file. This assumes the source image is in straight-alpha.
* The switch ``-m 1`` disables the generation of _mipmaps_ for the image. By default, the tool generates a full set of mipmaps when converting to a ``.dds``, but since our source image is not a power of two in width & height, it also generates a warning message about use with feature level 9.x devices. For standard sprites, we typically do not make use of _mipmaps_.
* The switch ``-f BC3_UNORM`` selects the ``DXGI_FORMAT_BC3_UNORM`` format for the resulting ``.dds`` file. In combination with the ``-pmalpha`` switch, this results in the "DXT4" [block-compression format](https://walbourn.github.io/direct3d-11-textures-and-block-compression/) being used.

# Rotating a sprite

Now that we have our cat rendering, we can start to animate it. Here's a simple rotation where we are using the **cosf** function to give us a time-varying value from -1 to 1.

In **Game.cpp**, modify the TODO of **Render**:

```cpp
float time = float(m_timer.GetTotalSeconds());

m_spriteBatch->Begin();

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::White, cosf(time) * 4.f, m_origin);

m_spriteBatch->End();
```

Build and run to see the cat spinning.

# Scaling a sprite

We can scale a sprite's size as well. Again, we are using **cosf** to give us a time-varying value between -1 and 1.

In **Game.cpp**, modify the TODO of **Render**:

```cpp
float time = float(m_timer.GetTotalSeconds());

m_spriteBatch->Begin();

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::White, 0.f, m_origin,
    cosf(time) + 2.f);

m_spriteBatch->End();
```

Build and run to see the cat growing and shrinking.

# Tinting a sprite

We can modify the color of the sprite with a tint as well:

In **Game.cpp**, modify the TODO of **Render**:

```cpp
m_spriteBatch->Begin();

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::Green, 0.f, m_origin);

m_spriteBatch->End();
```

Build and run to see a green-tinged cat.

# Tiling a sprite

With the optional source-rectangle parameter, we can tile a sprite.

In the **Game.h** file, add the following variable to the bottom of the Game class's private declarations:

```cpp
RECT m_tileRect;
```

In the **Game.cpp** file, modify in the TODO section of **CreateDeviceDependentResources**:

change

```cpp
m_origin.x = float(catDesc.Width / 2);
m_origin.y = float(catDesc.Height / 2);
```

to

```cpp
m_origin.x = float(catDesc.Width * 2);
m_origin.y = float(catDesc.Height * 2);

m_tileRect.left = catDesc.Width * 2;
m_tileRect.right = catDesc.Width * 6;
m_tileRect.top = catDesc.Height * 2;
m_tileRect.bottom = catDesc.Height * 6;
```

In the **Game.cpp** file, modify in the TODO section of **Render**:

```cpp
m_spriteBatch->Begin(SpriteSortMode_Deferred, nullptr,
      m_states->LinearWrap());

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, &m_tileRect,
    Colors::White, 0.f, m_origin);

m_spriteBatch->End();
```

Build and run to see the sprite as an array of 4x4 cats.

![Screenshot of cat sprite](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotSpriteCat3.PNG)

## Technical Notes

By default ``SpriteBatch`` uses a ``LinearClamp`` sampler. In order to do the tiling, we had to override that setting by providing a different ``LinearWrap`` sampler.

# Stretching a sprite

Using the optional destination rectangle instead of a 2D position, we can stretch a sprite.

In the **Game.h** file, add the following variable to the bottom of the Game class's private declarations:

```cpp
RECT m_stretchRect;
```

In the **Game.cpp** file, add to the TODO section of **CreateWindowSizeDependentResources**:

```cpp
m_stretchRect.left = size.right / 4;
m_stretchRect.top = size.bottom / 4;
m_stretchRect.right = m_stretchRect.left  + size.right / 2;
m_stretchRect.bottom = m_stretchRect.top + size.bottom / 2;
```

In the **Game.cpp** file, modify in the TODO section of **Render**:

```cpp
m_spriteBatch->Begin();

m_spriteBatch->Draw(m_texture.Get(), m_stretchRect, nullptr,
    Colors::White);

m_spriteBatch->End();
```

Build and run to see the sprite blown up

![Screenshot of cat sprite](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotSpriteCat4.PNG)

# Drawing a background image

Our last exercise for this lesson is rendering a sprite as a full background image.  Start by saving
[sunset.jpg](https://github.com/Microsoft/DirectXTK/wiki/images/sunset.jpg) to your project directory, and then from the top menu select **Project** / **Add Existing Item...**. Select "sunset.jpg" and click "OK".

In the **Game.h** file, add  the following variables to the bottom of the Game class's private declarations:

```cpp
RECT m_fullscreenRect;
Microsoft::WRL::ComPtr<ID3D11ShaderResourceView> m_background;
```

In **Game.cpp**, add to the TODO of **CreateDeviceDependentResources**:

```cpp
DX::ThrowIfFailed(
    CreateWICTextureFromFile(device, L"sunset.jpg", nullptr,
    m_background.ReleaseAndGetAddressOf()));
```

In **Game.cpp**, add to the TODO of **CreateWindowSizeDependentResources**:

```cpp
m_fullscreenRect = m_deviceResources->GetOutputSize();
```

nd then modify the ``m_origin`` initialization back to:

```cpp
m_origin.x = float(catDesc.Width / 2);
m_origin.y = float(catDesc.Height / 2);
```

In **Game.cpp**, add to the TODO of **OnDeviceLost**:

```cpp
m_background.Reset();
```

In **Game.cpp**, modify the TODO section of **Render** to be:

```cpp
m_spriteBatch->Begin();

m_spriteBatch->Draw(m_background.Get(), m_fullscreenRect);

m_spriteBatch->Draw(m_texture.Get(), m_screenPos, nullptr,
    Colors::White, 0.f, m_origin);

m_spriteBatch->End();
```

Build and run to see our cat drawing over a sunset background.

![Screenshot of cat sprite](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotSpriteCat5.PNG)

## Technical note

If we were only drawing the background (i.e. a single 'sprite' that fills an entire viewport), then using [[BasicPostProcess]]'s Copy operation is faster than ``SpriteBatch``, but requires Direct3D Hardware Feature Level 10.0 or later.

**Next lesson:** [[More tricks with sprites]]

# Further reading

DirectX Tool Kit docs [[CommonStates]], [[DDSTextureLoader]], [[SpriteBatch]], [[WICTextureLoader]]

[Direc3D 11 Textures and Block Compression](https://walbourn.github.io/direct3d-11-textures-and-block-compression/)  

[Premultiplied alpha](http://www.shawnhargreaves.com/blog/premultiplied-alpha.html)  
[Premultiplied alpha and image composition](http://www.shawnhargreaves.com/blog/premultiplied-alpha-and-image-composition.html)  
[Premultiplied alpha in XNA Game Studio 4.0](http://www.shawnhargreaves.com/blog/premultiplied-alpha-in-xna-game-studio-4-0.html)
