# fraymakers-text-generation
## Prerequisites - PLEASE READ
[Download the required files](https://github.com/shifterbit/fraymakers-text-generation/releases/tag/1.0)
- Add `dumProj.entity` and `text.entity` to your entity folder
- add `empty.png` and `empty.png.meta`   to wherever you put your sprites in your project, make sure they're right next to each other
- add base.png and base.png.meta to wherever you put your sprites in your project, make sure they're right next to each other
- extract  the contents of `font.zip` into wherever you put your sprites in your project
- extract the contents of `dummyProj.zip` into its own subfolder within your scripts folder in your project

And finally, ensure the following entry exists within your manifest file
```json
    {
      "id": "dummyProj",
      "type": "projectile",
      "objectStatsId": "dummyProjStats",
      "animationStatsId": "dummyProjAnimationStats",
      "hitboxStatsId": "dummyProjHitboxStats",
      "scriptId": "dummyProjScript",
      "costumesId": "assisttemplateCostumes"
    },
  ```




## The Code
At the in your script file add the following variable, make sure it is outside any functons such as `initialize()`, `update()` `onTeardown()` etc

```haxe
var globalDummy: Projectile = null;
```

```haxe
function createSpriteFromCharacter(char: String): Sprite {
    var res = getContent("text");
    var lowerCase = "abcdefghijklmnopqrstuvwxyz";
    var upperCase = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    var digits = "0123456789";
    var symbols = "!\"#$%&'()*+,-./;<=>?@:[\\]^_`{|}~ ";

    var lowerCaseIndex = lowerCase.indexOf(char);
    var upperCaseIndex = upperCase.indexOf(char);
    var digitIndex = digits.indexOf(char);
    var symbolIndex = symbols.indexOf(char);

    var isDigit = digitIndex >= 0;
    var isLowerCase = lowerCaseIndex >= 0;
    var isUpperCase = upperCaseIndex >= 0;
    var isSymbol = symbolIndex >= 0;

    var sprite: Sprite = Sprite.create(res);
    if (isDigit) {
        sprite.currentAnimation = "digits";
        sprite.currentFrame = digitIndex + 1;
    } else if (isSymbol) {
        sprite.currentAnimation = "symbols";
        sprite.currentFrame = symbolIndex + 1;
    } else if (isLowerCase) {
        sprite.currentAnimation = "lowercase";
        sprite.currentFrame = lowerCaseIndex + 1;
    } else if (isUpperCase) {
        sprite.currentAnimation = "uppercase";
        sprite.currentFrame = upperCaseIndex + 1;
    } else {
        sprite.currentAnimation = "symbols";
        sprite.currentFrame = symbols.length - 1;
    }

    return sprite;
}

function parseHexCode(hex: String): Int {
    var total: Int = 0;
    hex = hex.toUpperCase();
    var digits = "0123456789ABCDEF";
    var numPart = hex.substr(1);
    var idx = 0;
    while (idx < numPart.length) {
        var power = numPart.length - idx - 1;
        var num = digits.indexOf(numPart.charAt(idx));
        total += Math.round(num * Math.pow(16, power));
        idx++;
    }
    return total;
}

function syntaxError(text: String, curr: Int): { text: String, color: Int, error: Bool, length: Int } {
    var errorMessage: String = [
        "Error Rendering Text: \nUnexpected character ",
        ("\"" + text.charAt(curr) + "\""),
        "\nat character position ", curr.toString()].join("");
    Engine.log(errorMessage, 0xFFC300);
    return {
        error: true,
        color: 0xFF0000,
        text: errorMessage
    };
}

function parseTag(text: String, curr: Int): { text: String, color: Int, error: Bool, length: Int } {
    var color = 0;
    var content = "";

    if ("<" == text.charAt(curr)) { curr++; } else { return syntaxError(text, curr); }

    while (text.charAt(curr) == " ") { curr++; }

    if (text.charAt(curr) == "#") {
        var hexString = "#";
        var nums = "0123456789ABCDEFabcdef";
        curr++;
        while (nums.indexOf(text.charAt(curr)) >= 0) {
            hexString += text.charAt(curr);
            curr++;
        }
        color = parseHexCode(hexString);
    } else { return syntaxError(text, curr); }

    while (text.charAt(curr) == " ") { curr++; }

    if (text.charAt(curr) == ">") { curr++; } else { return syntaxError(text, curr); }

    while (text.charAt(curr) != "<") {
        if (text.charAt(curr) == "\\" && curr + 2 < text.length) {
            content += text.charAt(curr + 1);
            curr += 2;
        } else {
            content += text.charAt(curr);
            curr++;
        }
    }

    if (text.charAt(curr) == "<") { curr++; } else { return syntaxError(text, curr); };

    if (text.charAt(curr) == "/") { curr++; } else { return syntaxError(text, curr); };

    if (text.charAt(curr) == ">") { curr++; } else { return syntaxError(text, curr); };

    return { color: color, text: content, length: curr };
}

function parseText(text: String): Array<{ text: String, color: Int, error: Bool, length: Int }> {
    var nodes: Array<{ text: String, color: Int, error: Bool, length: Int }> = [];
    var curr = 0;
    var nodeIdx = 0;
    while (curr < text.length) {
        if (nodes[nodeIdx] == null) {
            nodes[nodeIdx] = {
                text: "",
                color: 0xFFFFFF,
                length: 0,
                error: false
            };
        }

        switch (text.charAt(curr)) {
            case "\\": {
                if (curr + 2 < text.length) {
                    nodes[nodeIdx].text += text.charAt(curr + 1);
                    nodes[nodeIdx].length += 1;
                    curr += 2;
                } else {
                    nodes.push(syntaxError(text, curr));
                    break;
                }
            };
            case "<": {
                nodeIdx++;
                var result = parseTag(text, curr);
                if (!result.error) {
                    curr = result.length;
                    nodes[nodeIdx] = result;
                    nodes[nodeIdx].color = result.color;
                    nodeIdx++;
                } else {
                    nodes[nodeIdx] = result;
                    break;
                }
            };
            default: {
                nodes[nodeIdx].text += text.charAt(curr);
                nodes[nodeIdx].length += 1;
                curr++;
            };

        }
    }
    return nodes;
}

function renderText(
    text: String,
    sprites: Array<Sprite>,
    container: Container,
    options: { autoLinewrap: Int, delay: Int, x: Int, y: Int }
): { duration: Int, sprites: Array<Sprite> } {
    var parsed: Array<{ text: String, color: Float, error: Boolean, length: number }> = parseText(text);
    Engine.forEach(sprites, function (sprite: Sprite, idx: Int) {
        sprite.dispose();
        return true;
    }, []);

    sprites = [];
    var line = 0;
    var col = 0;

    function makeSprite(char: String, shaderOptions: { color: Int }) {
        if (options != null && options.autoLinewrap > 0 && options.autoLinewrap < col && !options.wordWrap) {
            col = 0;
            line++;
        }
        var sprite: Sprite = createSpriteFromCharacter(char);
        sprite.x = col * 6;
        if (options != null && options.x != null) {
            sprite.x += options.x;
        }
        sprite.y = line * 10;
        if (options != null && options.y != null) {
            sprite.y += options.y;
        }
        var shader: RgbaColorShader = createColorShader(shaderOptions.color);
        sprite.addShader(shader);
        return sprite;
    }

    Engine.forEach(parsed, function (node: {
        text: String, color: Int, error: Boolean, length: number
    }, _: Int) {
        Engine.forCount(node.text.length, function (idx: Int) {
            var char: String = node.text.charAt(idx);
            if (char == "\n") {
                line++;
                col = 0;
            } else {
                var sprite = makeSprite(char, { color: node.color });
                sprites.push(sprite);
                col++;
            }
            return true;
        }, []);
        return true;
    }, []);

    if (globalDummy == null) {
        var dummy: Projectile = match.createProjectile(getContent("dummyProj"), null);
        globalDummy = dummy;
    }
    
    if (options.delay != null && options.delay > 0) {
        Engine.forEach(sprites, function (sprite: Sprite, idx: Int) {
            globalDummy.addTimer(idx * options.delay, 1, function () {
                container.addChild(sprite);
            }, { persistent: true });
            return true;
        }, []);

        return { sprites: sprites, duration: sprites.length * options.delay };
    } else {
        Engine.forEach(sprites, function (sprite: Sprite, _idx: Int) {
            container.addChild(sprite);
            return true;
        }, []);

        return { sprites: sprites, duration: 0 };
    }
}

function renderLines(lines: Array<String>,
    sprites: Array<Sprite>,
    container: Container,
    options: { delay: Int, x: Int, y: Int }): { duration: Int, sprites: Array<Sprite> } {
    var renderData = renderText(lines.join("\n"), sprites, container, { autoLinewrap: false, delay: delay, x: options.x, y: options.y });
    return renderData;
}
```

## How to use this
First you need to create an empty array of sprites
you need one for each individual body of text you wish to manage, otherwise it'll get overriden
So if you want a seperate body of text you will have to create a new array corresponding to it and reference it later;
```haxe
var textSprites: Array<Sprites> = [];
```
### Regular calls
To render text you just call `renderText` like so:
In this case we'll just render it starting at the center of the screen
```haxe

var renderData = renderText("Hey I'm displaying text on screen!",
                            textSprites, camera.getForegroundContainer(),
                            {
                              x: camera.getViewportWidth() / 2,
                              y: camera.getViewportHeight() / 2,
                            });
```

if you want the text to be animated, as in render character by character you can also set the delay argument, 
which is the frame delay between each character render:
```haxe

var renderData = renderText("Hey I'm displaying text on screen!",
                            textSprites, camera.getForegroundContainer(),
                            {
                              x: camera.getViewportWidth() / 2,
                              y: camera.getViewportHeight() / 2,
                              delay: 3
                            });
```
If you want newlines to automatically be created given a character count
```haxe
var renderData = renderText("Hey I'm displaying text on screen! Also this this might proabably cause a line break don't you think?",
                            textSprites, camera.getForegroundContainer(),
                            {
                              x: camera.getViewportWidth() / 2,
                              y: camera.getViewportHeight() / 2,
                              delay: 3,
                              autoLineWrap: 40
                            });
```
You can also use the neat `renderLines` function to do that for you
```haxe
var renderData = renderLines(["This will be the first line", "This will be the second line", "This is line three!"],
                            textSprites, camera.getForegroundContainer(),
                            {
                              x: camera.getViewportWidth() / 2,
                              y: camera.getViewportHeight() / 2,
                              delay: 3,
                            });
```

## Erasing Text
Erasing text works as follows
```haxe
Engine.forEach(sprites, function (sprite: Sprite, _idx: Int) {
        sprite.dispose();
        return true;
    }, []);
```
### Erasing text after rendering
Howerver if you want to erase text a certain time after it has rendered you can make use of timer as well as use the data returned by `renderLines` or `renderText`
like so:
```haxe
var renderData = renderLines(["This will be the first line", "This will be the second line", "This is line three!"],
                            textSprites, camera.getForegroundContainer(),
                            {
                              x: camera.getViewportWidth() / 2,
                              y: camera.getViewportHeight() / 2,
                              delay: 3,
                            });

globalDummy.addTimer(renderData.duration + 90, 1, function () {
        Engine.forEach(sprites, function (sprite: Sprite, idx: Int) {
            sprite.dispose();
            return true;
        }, []);
    }, { persistent: true });

```
This will erase the text 90 frames after it has been completely rendered.

## Displaying Text on top of a vfx
```haxe
var sprites: Array<Sprite> = [];
var vfx: Vfx = match.createVfx(new VfxStats({
        x: camera.getViewportWidth() / 2,
        y: camera.getViewportHeight() / 2,
        spriteContent: self.getResource().getContent("text"),
        animation: "base",
        loop: true
 }), null);
vfx.setX(vfx.getX() - vfx.getSprite().width / 2); // This ensures the textbos is in the middle
camera.getForegroundContainer().addChildAt(vfx.getViewRootContainer(), 0);
var renderData = renderText(mission, sprites, vfx.getViewRootContainer(), {
        delay: 3, autoLinewrap: 40,
        x: 0, y: 0
    });
sprites = renderData.sprites;
globalDummy.addTimer(renderData.duration + 90, 1, function () {
        Engine.forEach(sprites, function (sprite: Sprite, idx: Int) {
            sprite.dispose();
            return true;
        }, []);
        vfx.dispose();
        vfx.kill();
  }, { persistent: true });

```
For reference, this is where text would start rendering relative to the vfx
![image](https://github.com/user-attachments/assets/365030a6-f2fb-4eb4-ae57-37dc05c9d5f0)

## Text Formatting syntax and Newlines
The text render also supports basic color formatting via hex codes, and the syntax works as follows:
```
var formttedString = "<#HEXCODE>Text Body Here</>";
```
replacing `#HEXCODE` with a hexcode such as `#FF0000`, you can have as many formatting tags as you want, but nesting them will not work like 
```<#FFFFFF>Text <#FF0000>Inner</> </>```, so don't do that.
What you can do instead is just have tags one after another like:
```txt
<#FF0000> I'm red</> Now I'm not <#0000FF>Now I'm Feeling Blue</> <#00FF00>but no so green</>
```
If there's a syntax error in your formatting such as
```txt
<obviouslywrong>Test</>
```
An error would be displayed in the logs(only in training mode), but also within the rendered text itself, so take care when writing themm







