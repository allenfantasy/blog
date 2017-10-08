title: Notes about ... color

tags:
- FE
- color
---
When studying the CSS framework [Bluma](bluma.io), I found that it use a very smart way to determine the text color based on the bg color, utiliizing the `color luminance` to decide to use black text or white text, which is interesting and I found some interesting facts to note it down.
<!--more-->

### Color Luminance

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

### HSL

[HSL和HSV色彩空间](https://zh.wikipedia.org/wiki/HSL%E5%92%8CHSV%E8%89%B2%E5%BD%A9%E7%A9%BA%E9%97%B4#.E4.BB.8EHSL.E5.88.B0RGB.E7.9A.84.E8.BD.AC.E6.8D.A2)

> HSL和HSV都是一种将RGB色彩模型中的点在圆柱坐标系中的表示法。这两种表示法试图做到比RGB基于笛卡尔坐标系的几何结构更加直观。

![](https://upload.wikimedia.org/wikipedia/commons/thumb/a/a0/Hsl-hsv_models.svg/800px-Hsl-hsv_models.svg.png)

* Hue（H），色相，是色彩的基本属性，即平时所说的颜色名称
* Saturation（S），饱和度，是指色彩的纯度，越高色彩越纯，低则逐渐变灰，取 0~100% 的数值
* Lightness（L），亮度，取 0~100%

### RGB => HSL

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