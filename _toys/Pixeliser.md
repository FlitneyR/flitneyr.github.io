---
title: Pixeliser
---

## Pixeliser

<div>
<label>Text</label>
<input
    type="text"
    id="input"
    onkeyup="update_output();"
    />
</div>

<div>
<label>Off Pixel</label>
<input
    type="text"
    id="offpixel"
    onkeyup="update_output();"
    />
</div>

<div>
<label>On Pixel</label>
<input
    type="text"
    id="onpixel"
    onkeyup="update_output();"
    />
</div>

<button onclick="copy_to_clipboard();">Copy</button>

<div id="output" style="font-family: 'Noto Sans Mono', monospace"></div>

<script>
    var input = document.getElementById("input");
    var output = document.getElementById("output");

    const copy_to_clipboard = function() {
        navigator.clipboard.writeText(output.innerText);
    }

    const update_output = function () {
        var text = input.value;

        const pixels = [
            document.getElementById("offpixel").value,
            document.getElementById("onpixel").value,
        ];

        var result = [
            [],
            [],
            [],
            [],
            [],
            [],
            [],
        ];

        text = text.toUpperCase();

        for (let idx = 0; idx < text.length; ++idx) {
            pattern = keymap[text.charAt(idx)];

            if (!pattern) {
                continue;
            }
                
            if (idx == 0) {
                for (let row = 0; row < result.length; ++row)
                {
                    result[row].push(0);
                }
            }

            result[0].push(...Array(pattern[0].length).fill(0));
            result[result.length-1].push(...Array(pattern[0].length).fill(0));

            for (let row = 0; row < pattern.length; ++row) {
                result[row + 1].push(...pattern[row]);
            }

            if (idx + 1 < text.length)
            {
                for (let row = 0; row < result.length; ++row) {
                    result[row].push(0);
                }
            }
            else
            {
                for (let row = 0; row < result.length; ++row)
                {
                    result[row].push(0);
                }
            }
        }

        output.innerHTML = "";

        for (let row = 0; row < result.length; ++row)
        {
            for (let col = 0; col < result[row].length; ++col)
            {
                output.innerHTML += pixels[result[row][col]];
            }

            output.innerHTML += "<br>";
        }
    }

    const pixels = [
        " :whitemoss: ",
        " :mosswall: "
    ];

    const keymap = {
        "A": [
            [ 0, 1, 0 ],
            [ 1, 0, 1 ],
            [ 1, 1, 1 ],
            [ 1, 0, 1 ],
            [ 1, 0, 1 ],
        ],
        "B": [
            [ 1, 1, 0 ],
            [ 1, 0, 1 ],
            [ 1, 1, 0 ],
            [ 1, 0, 1 ],
            [ 1, 1, 0 ],
        ],
        "C": [
            [ 0, 1, 1 ],
            [ 1, 0, 0 ],
            [ 1, 0, 0 ],
            [ 1, 0, 0 ],
            [ 0, 1, 1 ],
        ],
        "D": [
            [ 1, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 1, 1, 0 ],
        ],
        "E": [
            [ 1, 1, 1 ],
            [ 1, 0, 0 ],
            [ 1, 1, 0 ],
            [ 1, 0, 0 ],
            [ 1, 1, 1 ],
        ],
        "F": [
            [ 1, 1, 1 ],
            [ 1, 0, 0 ],
            [ 1, 1, 0 ],
            [ 1, 0, 0 ],
            [ 1, 0, 0 ],
        ],
        "G": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 0 ],
            [ 1, 0, 1, 1 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        "H": [
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 1, 1, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
        ],
        "I": [
            [ 1, 1, 1 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
            [ 1, 1, 1 ],
        ],
        "J": [
            [ 1, 1, 1 ],
            [ 0, 0, 1 ],
            [ 0, 0, 1 ],
            [ 0, 0, 1 ],
            [ 1, 1, 0 ],
        ],
        "K": [
            [ 1, 0, 1 ],
            [ 1, 0, 1 ],
            [ 1, 1, 0 ],
            [ 1, 0, 1 ],
            [ 1, 0, 1 ],
        ],
        "L": [
            [ 1, 0, 0 ],
            [ 1, 0, 0 ],
            [ 1, 0, 0 ],
            [ 1, 0, 0 ],
            [ 1, 1, 1 ],
        ],
        "M": [
            [ 1, 0, 0, 0, 1 ],
            [ 1, 1, 0, 1, 1 ],
            [ 1, 0, 1, 0, 1 ],
            [ 1, 0, 0, 0, 1 ],
            [ 1, 0, 0, 0, 1 ],
        ],
        "N": [
            [ 1, 0, 0, 1 ],
            [ 1, 1, 0, 1 ],
            [ 1, 0, 1, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
        ],
        "O": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        "P": [
            [ 1, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 1, 1, 1, 0 ],
            [ 1, 0, 0, 0 ],
            [ 1, 0, 0, 0 ],
        ],
        "Q": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
            [ 0, 0, 0, 1 ],
        ],
        "R": [
            [ 1, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 1, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
        ],
        "S": [
            [ 0, 1, 1, 1 ],
            [ 1, 0, 0, 0 ],
            [ 0, 1, 1, 0 ],
            [ 0, 0, 0, 1 ],
            [ 1, 1, 1, 0 ],
        ],
        "T": [
            [ 1, 1, 1 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
        ],
        "U": [
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        "V": [
            [ 1, 0, 0, 0, 1 ],
            [ 1, 0, 0, 0, 1 ],
            [ 1, 0, 0, 0, 1 ],
            [ 0, 1, 0, 1, 0 ],
            [ 0, 0, 1, 0, 0 ],
        ],
        "W": [
            [ 1, 0, 0, 0, 1 ],
            [ 1, 0, 0, 0, 1 ],
            [ 1, 0, 1, 0, 1 ],
            [ 1, 1, 0, 1, 1 ],
            [ 1, 0, 0, 0, 1 ],
        ],
        "X": [
            [ 1, 0, 0, 0, 1 ],
            [ 0, 1, 0, 1, 0 ],
            [ 0, 0, 1, 0, 0 ],
            [ 0, 1, 0, 1, 0 ],
            [ 1, 0, 0, 0, 1 ],
        ],
        "Y": [
            [ 1, 0, 1 ],
            [ 1, 0, 1 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
        ],
        "Z": [
            [ 1, 1, 1, 1 ],
            [ 0, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 0 ],
            [ 1, 1, 1, 1 ],
        ],
        "0": [
            [ 0, 1, 0 ],
            [ 1, 0, 1 ],
            [ 1, 0, 1 ],
            [ 1, 0, 1 ],
            [ 0, 1, 0 ],
        ],
        "1": [
            [ 0, 1, 0 ],
            [ 1, 1, 0 ],
            [ 0, 1, 0 ],
            [ 0, 1, 0 ],
            [ 1, 1, 1 ],
        ],
        "2": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 0, 0, 1 ],
            [ 0, 0, 1, 0 ],
            [ 1, 1, 1, 1 ],
        ],
        "3": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 0, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        "4": [
            [ 0, 0, 1, 0 ],
            [ 0, 1, 1, 0 ],
            [ 1, 0, 1, 0 ],
            [ 1, 1, 1, 1 ],
            [ 0, 0, 1, 0 ],
        ],
        "5": [
            [ 1, 1, 1, 1 ],
            [ 1, 0, 0, 0 ],
            [ 1, 1, 1, 0 ],
            [ 0, 0, 0, 1 ],
            [ 1, 1, 1, 0 ],
        ],
        "6": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 0 ],
            [ 1, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        "7": [
            [ 1, 1, 1, 1 ],
            [ 0, 0, 0, 1 ],
            [ 0, 0, 1, 0 ],
            [ 0, 1, 0, 0 ],
            [ 1, 0, 0, 0 ],
        ],
        "8": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        "9": [
            [ 0, 1, 1, 0 ],
            [ 1, 0, 0, 1 ],
            [ 0, 1, 1, 1 ],
            [ 0, 0, 0, 1 ],
            [ 0, 1, 1, 0 ],
        ],
        " ": [
            [ 0 ],
            [ 0 ],
            [ 0 ],
            [ 0 ],
            [ 0 ],
        ],
        "'": [
            [ 1 ],
            [ 1 ],
            [ 0 ],
            [ 0 ],
            [ 0 ],
        ],
        "!": [
            [ 1 ],
            [ 1 ],
            [ 1 ],
            [ 0 ],
            [ 1 ],
        ],
        "?": [
            [ 1, 1, 0 ],
            [ 0, 0, 1 ],
            [ 0, 1, 0 ],
            [ 0, 0, 0 ],
            [ 0, 1, 0 ],
        ]
    };
</script>
