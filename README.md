Generate css background gradient based on the color of some image.



This is the code,

```js
import ColorThief from "./UI/ColorThief";

function adjustBrightnessFromRGB(arrRGB = [0, 0, 0], amount = 10) {
    // to make the colour less bright than the input
    // change the following three "+" symbols to "-"
    var R = arrRGB[0] + amount;
    var G = arrRGB[1] + amount;
    var B = arrRGB[2] + amount;

    if (R > 255) R = 255;
    else if (R < 0) R = 0;

    if (G > 255) G = 255;
    else if (G < 0) G = 0;

    if (B > 255) B = 255;
    else if (B < 0) B = 0;

    return [R, G, B];
}

function getBrightnessFromRGB(arrRGB) {
    var luma = 0.2126 * arrRGB[0] + 0.7152 * arrRGB[1] + 0.0722 * arrRGB[2]; // per ITU-R BT.709
    return luma;
}

export function getDynamicTheme(
    imgElement,
    light = 100,
    noFontColor,
    colorSampling = 6,
    contrastThreshold = 166
) {
    var takeColor = 4;
    var CT = new ColorThief();
    var palettes = CT.getPalette(imgElement, colorSampling);

    var arr = [];
    palettes.forEach((element) => {
        // Only pick bright color
        if (getBrightnessFromRGB(element) > light) {
            arr.push(element);
        }
    });

    // Default value or Fallback
    var output = [];
    var sumBrightness = 0;
    for (let i = 0; i < takeColor; i++) {
        var theColor = arr[i]
            ? arr[i]
            : palettes[i]
            ? palettes[i]
            : [255, 255, 255];

        sumBrightness += getBrightnessFromRGB(theColor);
        output.push(theColor);
    }

    // the average color is dark or light
    var avgBrightness = output.length == 0 ? 0 : sumBrightness / output.length;
    // Contrast color for the text / font
    var contrastColor =
        avgBrightness <= contrastThreshold ? "#FFFFFF" : "#000000";

    // If too dark
    if (sumBrightness < 10) {
        let startBrightness = 60;
        for (let i = 0, len = output.length; i < len; i++) {
            output[i] = adjustBrightnessFromRGB(output[i], startBrightness);
            startBrightness += 30;
        }
    }

    // Calculate the difference if the color is similar then make brighter background,
    // so the origin image can be seen
    function deltaE(rgbA, rgbB) {
        let r = Math.abs(rgbA[0] - rgbB[0]);
        let g = Math.abs(rgbA[1] - rgbB[1]);
        let b = Math.abs(rgbA[2] - rgbB[2]);
        return r + g + b;
    }

    var diff = 0;
    for (let i = 0, len = output.length - 1; i < len; i++) {
        diff += deltaE(output[i], output[i + 1]);
    }

    // the different less than threshold
    if (diff < 100) {
        // Change to darken or brighten
        let startBrightness = 40;

        if (contrastColor == "#000000") {
            startBrightness = -40;
        }

        for (let i = 0, len = output.length; i < len; i++) {
            output[i] = adjustBrightnessFromRGB(output[i], startBrightness);

            startBrightness += 10;
        }
    }

    function formatRGB(arrRGB, alpha = 1) {
        return `rgba(${arrRGB[0]},${arrRGB[1]},${arrRGB[2]},${alpha})`;
    }

    return {
        backgroundColor: output[0],
        backgroundImage: `linear-gradient(${getRandomInt(
            30,
            60
        )}deg, ${formatRGB(output[0])} 0%,${formatRGB(
            output[1]
        )} 46%, ${formatRGB(output[2])} 100%)`,
        ...(noFontColor ? {} : { color: contrastColor }),
    };
}
```


Make some react hooks (if you are using react)
```js 
export function useDynamicTheme(src, light = 100, noFontColor = false) {
    const [color, setColor] = useState();
    const isMountedRef = useIsMountedRef();

    const getCardTheme = useCallback(
        async (src, light, noFontColor) => {
            var im = new Image();
            im.src = src;
            im.onload = () => {
                if (isMountedRef.current) {
                    let a = getDynamicTheme(im, light, noFontColor);
                    setColor(a);
                }
            };
        },
        [isMountedRef]
    );

    useEffect(() => {
        if (src) {
            getCardTheme(src, light, noFontColor);
        } else {
            setColor(null);
        }
    }, [src, light, noFontColor]);

    return color;
}
```

This is the themed card based on image

```js
import { Card } from "@mui/material";
import { useDynamicTheme } from "../Helper";

export default function CardThemed({
    src,
    sx = {},
    disabled = false,
    light = 100,
    noFontColor = false,
    children,
    ...rest
}) {
    const color = useDynamicTheme(src, light, noFontColor);

    return (
        <Card
            sx={{
                ...(!disabled ? (color ? color : {}) : {}),
                ...sx,
                border: "none",
                boxShadow: "none",
            }}
            {...rest}
        >
            {children}
        </Card>
    );
}
```