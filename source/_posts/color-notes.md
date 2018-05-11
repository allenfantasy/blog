title: Color, the known and unknown
date: 2017-10-08 16:43
updated: 2017-10-23 11:24

tags:
- Color
---
When studying the CSS framework [Bluma](bluma.io), I found that it use a very smart way to determine the text color based on the bg color, utiliizing the `color luminance` to decide to use black text or white text, which is interesting and I found some interesting facts to note it down.
<!--more-->

### Color Luminance

Steven Bradley wrote an great article about [color luminance](http://vanseodesign.com/web-design/color-luminance/), and here are some details:

1. Our eyes have rods（视杆细胞）and cones（视锥细胞）to perceive light/dark as well as different color information; rods are responsible for seeing in low light and are sensitive to light/dark, while cones are responsible for distinguishing different colors. Moreover we have S-cones (blue), M-cones (green) and L-cones (red) for short, medium and long wavelength. -- three primary colors.
2. `Luminance` is the measurement of the intensity of light that reaches our eye, while `brightness` and `value` are only the perception of an object's luminance. `Lightness` is the brightness relative to the brightness of a similarly illuminated white.
3. Our perception of lightness (or brightness) don’t scale linearly with luminance. The perceived luminance is dependent on both light intensity and the specific wavelength of that light (a.k.a type of color).
4. Every color has its own natural luminance levels.
5. Saturation also affects luminance. If you reduce the saturation of a pure color to 0% the result is a 50% grey with a 50% value for luminance.
6. differences between HSL, HSB, and HSV:
    * HSB/V is measuring **the amount of light**
    * HSL is measuring **the amount of white**

#### Quantitive Implementation

From [WCAG's Definition](http://www.w3.org/TR/2008/REC-WCAG20-20081211/#relativeluminancedef), the `relative luminance` is:

> the relative brightness of any point in a colorspace, normalized to 0 for darkest black and 1 for lightest white.

WCAG also provides the formula for luminance:

> For the sRGB colorspace, the relative luminance of a color is defined as `L = 0.2126 * R + 0.7152 * G + 0.0722 * B` where R, G and B are defined.
> 
> as:
> 
> * if RsRGB <= 0.03928 then R = RsRGB/12.92 else R = ((RsRGB+0.055)/1.055) ^ 2.4
> * if GsRGB <= 0.03928 then G = GsRGB/12.92 else G = ((GsRGB+0.055)/1.055) ^ 2.4
> * if BsRGB <= 0.03928 then B = BsRGB/12.92 else B = ((BsRGB+0.055)/1.055) ^ 2.4
> 
> and RsRGB, GsRGB, and BsRGB are defined as:
> 
> * RsRGB = R8bit / 255
> * GsRGB = G8bit / 255
> * BsRGB = B8bit / 255


[Bluma](bluma.io) strictly implements the spec above by having such Sass function:

```sass
@function colorLuminance($color)
  $color-rgb: ('red': red($color),'green': green($color),'blue': blue($color))
  @each $name, $value in $color-rgb
    $adjusted: 0
    $value: $value / 255
    @if $value < 0.03928
      $value: $value / 12.92
    @else
      $value: ($value + .055) / 1.055
      $value: powerNumber($value, 2)
    $color-rgb: map-merge($color-rgb, ($name: $value))
  @return (map-get($color-rgb, 'red') * .2126) + (map-get($color-rgb, 'green') * .7152) + (map-get($color-rgb, 'blue') * .0722)
```

Another similar implementation is documented [here](https://css-tricks.com/snippets/sass/luminance-color-function/)

[Work With Color](http://www.workwithcolor.com/color-luminance-2233.htm) says: 

> Luminance on the other hand is a measure to describe the perceived brightness of a color
> 
> Contrast as in the distance of luminance between two colors
> 
> Color decisions need to consider luminance / contrast because it is key to usability.
[HSL Color Picker](http://www.workwithcolor.com/hsl-color-picker-01.htm)

From [CSS Tricks' Definition](https://css-tricks.com/snippets/sass/luminance-color-function/):

> To put it simply, the luminance of a color defines whether its brightness. A luminance of 1 means the color is white. On the opposite, a luminance score of 0 means the color is black.

NOTE: WCAG = Web Content Accessibility Guidelines

### This video blows my mind

<iframe width="560" height="315" src="https://www.youtube.com/embed/kVny7BswdqY" frameborder="0" allowfullscreen></iframe>

Notice 3:50 -- that is crazy. Remember that our eyes could be cheated.

### Color Models

[Colorizer](http://colorizer.org/) have listed all common color models.

---

#### RGB

From [Wikipedia](https://en.wikipedia.org/wiki/RGB_color_model):

> The **RGB color model** is an addictive color model in which red, green and blue light are added together in various ways to reproduce a broad array of colors.
> ...
> The RGB color model is *addictive* in the sense that the three light beams are added together, and their light spectra add, wavelength by wavelength, to make the final color's spectrum. This is essentially opposite to the *subtractive* color model that applies to paints, inks, dyes and other substances whose colors depends on *reflecting* the light under which we see them.
> ...

##### why R,G,B

> The choice of primary colors (red, green, blue) is related to the physiology of the human eye; good primaries are stimuli that maximize the difference between the responses of the cone cells of the human retina to light of different wavelengths, and that thereby make a large color triangle.
> ...
> The difference in the signals received from the three kinds allows the brain to differentiate a wide gamut of different colors.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/0/08/CIExy1931_sRGB_gamut_D65.png/440px-CIExy1931_sRGB_gamut_D65.png)

> The color triangle represents the range of colors which could be reproduced by additive mixing of non-negative amounts of three primary colors.

[为什么 “红、黄、蓝” 是三原色？而不是其他颜色？](https://www.zhihu.com/question/19646016)
[光的三原色和颜料的三原色不一样吗？为什么？](https://www.zhihu.com/question/23839549)

> “原色”的指定并没有唯一的选法，因为就理论上而言，凡是彼此之间无法替代的颜色都可以被选为“原色”，只是目前普遍认定“光的三原色”为红绿蓝。

##### numeric representations

* float
* percentage
* integer range (0,255) - decimal / hexadecimal
* larger integer ranges for each primary color (High-end digital image equipment)

**[Concept] illuminant**

From Wikipedia [Standard Illuminant](https://en.wikipedia.org/wiki/Standard_illuminant#Illuminant_series_D):

> The International Commission on Illumination (usually abbreviated CIE for its French name) is th body responsible for publishing all of the well-known standard illuminants.

There are many standards like Illuminant A, B, C ... The most common used one is called **Illuminant D**, which:

> ... represents phases of daylight, ... the D series of illuminants are constructed to represent natural daylight. They are difficult to produce artificially, but are easy to characterized mathematically.

##### What is white?

[White Point](https://en.wikipedia.org/wiki/White_point)

> A white point is a set of tristimulus values (a set of values of 3 primary colors) or chromaticity coordinates that *serve to define the color "white"* in image capture, encoding, or reproduction. Depending on the application, different definitions of white are needed to give acceptable results.

SPD = Spectral Power Distribution (光谱能量分布??)

* [D65](https://en.wikipedia.org/wiki/Illuminant_D65)
* [Subtractive color](https://zh.wikipedia.org/wiki/%E6%B8%9B%E8%89%B2%E6%B3%95)
* [Chromaticity](https://en.wikipedia.org/wiki/Chromaticity)

`色度(Colorfulness/Chroma/Saturation)`, `色相(Hue)`

*色度* 指的是色彩的纯度，也叫 *饱和度* 或者 *彩度*。

---

#### CMYK

[CMYK - Wikipedia](https://en.wikipedia.org/wiki/CMYK_color_model)

> The CMYK color model (process color, four color) is a subtractive color model, used in color printing, and is also used to describe the printing process itself.

* C - Cyan
* M - Magenta
* Y - Yellow
* K - Key (black)

#### HSL

[HSL和HSV色彩空间](https://zh.wikipedia.org/wiki/HSL%E5%92%8CHSV%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4#.E4.BB.8EHSL.E5.88.B0RGB.E7.9A.84.E8.BD.AC.E6.8D.A2)

> HSL和HSV都是一种将RGB色彩模型中的点在圆柱坐标系中的表示法。这两种表示法试图做到比RGB基于笛卡尔坐标系的几何结构更加直观。

![](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a0/Hsl-hsv_models.svg/800px-Hsl-hsv_models.svg.png)

Colorizer 使用了圆锥表示 HSL：

![](http://colorizer.org/img/hsl.png)

* Hue（H），色相，是色彩的基本属性，即平时所说的颜色名称
* Saturation（S），饱和度，是指色彩的纯度，越高色彩越纯，低则逐渐变灰，取 0~100% 的数值
* Lightness（L），亮度，取 0~100%

#### RGB => HSL

[Algorithm in CSS3 Specification](https://www.w3.org/TR/css3-color/#hsl-color)

[An JS implementation](https://stackoverflow.com/a/9493060/1301194) based on the spec:

```js
/**
 * Converts an HSL color value to RGB. Conversion formula
 * adapted from http://en.wikipedia.org/wiki/HSL_color_space.
 * Assumes h, s, and l are contained in the set [0, 1] and
 * returns r, g, and b in the set [0, 255].
 *
 * @param   {number}  h       The hue
 * @param   {number}  s       The saturation
 * @param   {number}  l       The lightness
 * @return  {Array}           The RGB representation
 */
function hslToRgb(h, s, l){
    var r, g, b;

    if(s == 0){
        r = g = b = l; // achromatic
    }else{
        var hue2rgb = function hue2rgb(p, q, t){
            if(t < 0) t += 1;
            if(t > 1) t -= 1;
            if(t < 1/6) return p + (q - p) * 6 * t;
            if(t < 1/2) return q;
            if(t < 2/3) return p + (q - p) * (2/3 - t) * 6;
            return p;
        }

        var q = l < 0.5 ? l * (1 + s) : l + s - l * s;
        var p = 2 * l - q;
        r = hue2rgb(p, q, h + 1/3);
        g = hue2rgb(p, q, h);
        b = hue2rgb(p, q, h - 1/3);
    }

    return [Math.round(r * 255), Math.round(g * 255), Math.round(b * 255)];
}
```

---

### Color Meaning

http://vanseodesign.com/web-design/color-meaning/

> ... there is no substantive evidence that support a universal system of color meaning. It’s not that colors themselves have specific meaning, but rather that **we have culturally assigned meanings to them**.

> ... it's important to understand who your target audience is and how your audience attaches meaning to color.

* **warm colors:** red, orange, yellow -- passion, energy, impulsiveness, happiness, coziness and comfort
* **cool colors:** green, blue, violet -- calm, trust, professionalism, also sadness and melancholy

### TODO

* HSB/HSV
* HSI (Hue, Saturation, Intensity) ???
* [Color Theory, The Color Wheel And Color Schemes](http://vanseodesign.com/web-design/color-theory/)
* [Whitespace: Less Is More In Web Design](http://vanseodesign.com/web-design/whitespace/)
* [Design Basics: Repetition To Create Visual Themes](http://vanseodesign.com/web-design/design-basics-repetition/)
* [Color Theory Basics](http://www.color-wheel-pro.com/color-theory-basics.html)
* [Basic techniques for combining colors](http://www.tigercolor.com/color-lab/color-theory/color-harmonies.htm)
* [Luminance is More Important than Color, The Refracted Light](http://therefractedlight.blogspot.jp/2010/06/luminance-is-more-important-than-color.html)
* [Color Systems — Part 1](http://vanseodesign.com/web-design/color-systems-1/)
* [Color Systems — Part 2](http://vanseodesign.com/web-design/color-systems-2/)