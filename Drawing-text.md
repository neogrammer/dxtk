|[[Getting Started]]|
|---|

This lesson covers drawing text using bitmap fonts and the sprite renderer.

> With the DirectX 11.1, you can also rely on Direct2D/DirectWrite being available which is recommended for true vector-font features such as high quality across a wide range of scales, for complex layouts, or large-alphabet fonts. SpriteFont is intended for low-overhead bitmap-font rendering using a font that can be captured to a single texture.

> Note that Windows Phone 8.0 and the Xbox XDK/GDK do not support Direct2D/DirectWrite. UWP on the Xbox device family does support it.

# Setup
First create a new project using the instructions from the earlier lessons: [[Using DeviceResources]] and
[[Adding the DirectX Tool Kit]] which we will use for this lesson.

# Creating a font

The text renderer in _DirectX Tool Kit_ makes use of bitmap fonts, so the first step is to create a ``.spritefont`` file by capturing a system TrueType font using the **makespritefont** command-line utility.

* Download the [MakeSpriteFont.exe](https://github.com/microsoft/DirectXTK/releases/latest/download/MakeSpriteFont.exe) from the _DirectX Tool Kit_ site save the EXE into your project's folder.
* Open a Command Prompt and then change to your project's folder.

Run the following command-line

    MakeSpriteFont "Courier New" myfile.spritefont /FontSize:32

Then from the top menu in Visual Studio select **Project** / **Add Existing Item...**. Select [myfile.spritefont](https://github.com/Microsoft/DirectXTK/wiki/myfile.spritefont) and click "OK".

> If you are using a Universal Windows Platform (UWP) app, Windows Store, or Xbox project rather than a Windows desktop app, you need to manually edit the Visual Studio project properties on the ``myfile.spritefont`` file and make sure "Content" is set to "Yes" so the data file will be included in your packaged build for *All Configurations* and *All Platforms*.

To get a **Bold** version of the font, run the following command-line:

    MakeSpriteFont "Courier New" myfileb.spritefont /FontSize:32 /FontStyle:Bold

For an _Italic_ version of the font, run the following command-line:

    MakeSpriteFont "Courier New" myfilei.spritefont /FontSize:32 /FontStyle:Italic

For a <strike>Strikeout</strike> version of the font, run the following command-line:

    MakeSpriteFont "Courier New" myfiles.spritefont /FontSize:32 /FontStyle:Strikeout

For a <u>Underline</u> version of the font, run the following command-line:

    MakeSpriteFont "Courier New" myfileu.spritefont /FontSize:32 /FontStyle:Underline

# Loading a bitmap font
In the **Game.h** file, add the following variable to the bottom of the Game class's private declarations:

```cpp
std::unique_ptr<DirectX::SpriteFont> m_font;
```

In **Game.cpp**, add to the TODO of **CreateDeviceDependentResources**:

```cpp
m_font = std::make_unique<SpriteFont>(device, L"myfile.spritefont");
```

In **Game.cpp**, add to the TODO of **OnDeviceLost**:

```cpp
m_font.reset();
```

Build and run the application which will still not be displaying anything but the cornflower blue window, but will have a font loaded.

<details><summary><i>Click here for troubleshooting advice</i></summary>
<p>If you get a runtime exception, then you may have the "myfile.spritefont" file in the wrong folder, have modified the "Working Directory" in the "Debugging" configuration settings, or otherwise changed the expected paths at runtime of the application. You should set a break-point on <code>std::make_unique&lt;SpriteFont&gt;(...)</code> and step into the code to find the exact problem.</p></details>

# Drawing text using a font

In the **Game.h** file, add the following variables to the bottom of the Game class's private declarations:

```cpp
DirectX::SimpleMath::Vector2 m_fontPos;
std::unique_ptr<DirectX::SpriteBatch> m_spriteBatch;
```

In **Game.cpp**, add to the TODO of **CreateDeviceDependentResources**:

```cpp
auto context = m_deviceResources->GetD3DDeviceContext();
m_spriteBatch = std::make_unique<SpriteBatch>(context);
```

In **Game.cpp**, add to the TODO of **CreateWindowSizeDependentResources**:

```cpp
auto size = m_deviceResources->GetOutputSize();
m_fontPos.x = float(size.right) / 2.f;
m_fontPos.y = float(size.bottom) / 2.f;
```

> If using the UWP template, you also need to add ``m_spriteBatch->SetRotation(m_outputRotation);`` to handle display orientation changes.

In **Game.cpp**, add to the TODO of **OnDeviceLost**:

```cpp
m_spriteBatch.reset();
```

In **Game.cpp**, modify the TODO section of **Render** to be:

```cpp
m_spriteBatch->Begin();

const wchar_t* output = L"Hello World";

Vector2 origin = m_font->MeasureString(output) / 2.f;

m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos, Colors::White, 0.f, origin);

m_spriteBatch->End();
```

Build and run to see our text string centered in the middle of the rendering window:

![Screenshot Hello World](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotHelloWorld.PNG)

# Using ``std::wstring`` for text

In **pch.h** after the other ``#include`` statements, add:

```cpp
#include <string>
```

In **Game.cpp**, modify the TODO section of **Render** to be:

```cpp
std::wstring output = std::wstring(L"Hello") + std::wstring(L" World");

m_spriteBatch->Begin();

Vector2 origin = m_font->MeasureString(output.c_str()) / 2.f;

m_font->DrawString(m_spriteBatch.get(), output.c_str(),
    m_fontPos, Colors::White, 0.f, origin);

m_spriteBatch->End();
```

Build and run to see our text string centered in the middle of the rendering window.

# Using ASCII text

In **pch.h** after the other ``#include`` statements, add:

```cpp
#include <codecvt>
#include <locale>
```

In **Game.cpp**, modify the TODO section of **Render** to be:

```cpp
const char *ascii = "Hello World";
std::wstring_convert<std::codecvt_utf8<wchar_t>> converter;
std::wstring output = converter.from_bytes(ascii);

m_spriteBatch->Begin();

Vector2 origin = m_font->MeasureString(output.c_str()) / 2.f;

m_font->DrawString(m_spriteBatch.get(), output.c_str(),
  m_fontPos, Colors::White, 0.f, origin);

m_spriteBatch->End();
```

Build and run to see our text string centered in the middle of the rendering window.

# Drop-shadow effect

In **Game.cpp**, modify the TODO section of **Render** to be:

```cpp
m_spriteBatch->Begin();

const wchar_t* output = L"Hello World";

Vector2 origin = m_font->MeasureString(output) / 2.f;

m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos + Vector2(1.f, 1.f), Colors::Black, 0.f, origin);
m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos + Vector2(-1.f, 1.f), Colors::Black, 0.f, origin);

m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos, Colors::White, 0.f, origin);

m_spriteBatch->End();
```

Build and run to see our text string centered in the middle of the rendering window but with a drop-shadow:

![Screenshot Hello World with shadow](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotHelloWorld2.PNG)

# Outline effect

In **Game.cpp**, modify the TODO section of **Render** to be:

```cpp
m_spriteBatch->Begin();

const wchar_t* output = L"Hello World";

Vector2 origin = m_font->MeasureString(output) / 2.f;

m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos + Vector2(1.f, 1.f), Colors::Black, 0.f, origin);
m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos + Vector2(-1.f, 1.f), Colors::Black, 0.f, origin);
m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos + Vector2(-1.f, -1.f), Colors::Black, 0.f, origin);
m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos + Vector2(1.f, -1.f), Colors::Black, 0.f, origin);

m_font->DrawString(m_spriteBatch.get(), output,
    m_fontPos, Colors::White, 0.f, origin);

m_spriteBatch->End();
```

Build and run to see our text string centered in the middle of the rendering window but with an outline:

![Screenshot Hello World with outline](https://github.com/Microsoft/DirectXTK/wiki/images/screenshotHelloWorld3.PNG)

# More to explore

* If you are wanting to render text similar to how a classic 'console' application does, see [[TextConsole]].

* If you want to render Xbox controller button artwork composed with standard text, see [[ControllerFont]].

* If you want to make use of *DirectWrite* for vector-based fonts on Windows, see [Microsoft Docs](https://docs.microsoft.com/windows/win32/directwrite/getting-started-with-directwrite).

**Next lesson:** [[Simple rendering]]

# Further reading

DirectX Tool Kit docs [[SpriteBatch]], [[SpriteFont]], [[MakeSpriteFont]]
