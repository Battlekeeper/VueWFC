<script setup lang="ts">
import GIF from 'gif.js';

var canvas = ref<HTMLCanvasElement>();
var canvasctx = canvas.value?.getContext("2d");
var board: Cell[][];
var stepsMade = 0;
var tiles: Tile[];
var tileSize = 20;
var bw = 32
var bh = 32

const imageCache: Record<string, HTMLImageElement> = {};
const drawnTiles: Set<string> = new Set();

var gif: any;


async function stepsim(render: boolean = false) {
    if (stepsMade < board.length * board[0].length) {
        stepsMade++;
        console.log("Step " + stepsMade);
        collapseStep();
        if (render) {
            await renderBoard();
        }
        return true
    }
    return false
}

async function autoPlay() {
    while (await stepsim(true)) { }
    console.log("Rendered")
    await renderBoard();
}


class Cell {
    row: number
    col: number
    path: string = ""
    possibleTiles: string[] = []
    constructor(row: number, col: number) {
        this.row = row;
        this.col = col;
    }
}
class Tile {
    path: string
    top: Uint8ClampedArray
    bottom: Uint8ClampedArray
    left: Uint8ClampedArray
    right: Uint8ClampedArray

    validTop: string[] = []
    validBottom: string[] = []
    validLeft: string[] = []
    validRight: string[] = []

    constructor(path: string, top: Uint8ClampedArray, bottom: Uint8ClampedArray, left: Uint8ClampedArray, right: Uint8ClampedArray) {
        this.path = path;
        this.top = top;
        this.bottom = bottom;
        this.left = left;
        this.right = right;
    }
}


async function getPixelData(imageUrl: string): Promise<Uint8ClampedArray | null> {
    return new Promise((resolve) => {
        const img = new Image();
        img.crossOrigin = "Anonymous"; // Enable CORS if the image is hosted on a different domain
        img.src = imageUrl;

        img.onload = () => {
            const canvas = document.createElement("canvas");
            const context = canvas.getContext("2d");

            if (!context) {
                resolve(null);
                return;
            }

            canvas.width = img.width;
            canvas.height = img.height;
            context.drawImage(img, 0, 0, img.width, img.height);

            const imageData = context.getImageData(0, 0, img.width, img.height).data;
            resolve(imageData);
            canvas.remove();
        };

        img.onerror = () => {
            resolve(null);
        };
    });
}
function areUint8ClampedArraysEqual(arr1: Uint8ClampedArray, arr2: Uint8ClampedArray): boolean {
    if (arr1.length !== arr2.length) {
        return false;
    }

    for (let i = 0; i < arr1.length; i++) {
        if (arr1[i] !== arr2[i]) {
            return false;
        }
    }

    return true;
}
async function loadTiles(path: string, fileNames: string[], size: number) {
    const tiles: Tile[] = [];
    for (const fileName of fileNames) {
        const data = await getPixelData(path + fileName);
        if (!data) {
            continue;
        }
        var top = data.slice(0, size * 4);
        var bottom = data.slice((size - 1) * size * 4, size * size * 4);
        var left = new Uint8ClampedArray(size * 4);
        var right = new Uint8ClampedArray(size * 4);
        for (let i = 0; i < size; i++) {
            const index = i * size * 4;
            left.set(data.slice(index, index + 4), i * 4);
            right.set(data.slice(index + (size - 1) * 4, index + size * 4), i * 4);
        }
        var tile = new Tile(fileName, top, bottom, left, right)
        tiles.push(tile);
    }
    return tiles;
}
function generateValid() {
    for (let index = 0; index < tiles.length; index++) {
        const element = tiles[index];
        for (let index2 = 0; index2 < tiles.length; index2++) {
            const element2 = tiles[index2];
            if (areUint8ClampedArraysEqual(element.top, element2.bottom)) {
                element.validTop.push(element2.path);
            }
            if (areUint8ClampedArraysEqual(element.bottom, element2.top)) {
                element.validBottom.push(element2.path);
            }
            if (areUint8ClampedArraysEqual(element.left, element2.right)) {
                element.validLeft.push(element2.path);
            }
            if (areUint8ClampedArraysEqual(element.right, element2.left)) {
                element.validRight.push(element2.path);
            }
        }
    }
}
function generateBoard(width: number, height: number) {
    var board: Cell[][] = [];
    for (let rowi = 0; rowi < height; rowi++) {
        var row: Cell[] = [];
        for (let coli = 0; coli < width; coli++) {
            var cell = new Cell(rowi, coli);
            cell.possibleTiles = tiles.map(x => x.path);
            row.push(cell);
        }
        board.push(row);
    }
    return board;
}
function findLowestEntropy() {
    var lowestEntropy = 100000;
    var lowestEntropyCells: Cell[] = [];
    for (let x = 0; x < board.length; x++) {
        const row = board[x];
        for (let y = 0; y < row.length; y++) {
            const cell = row[y];
            if (cell.path.length == 0) {
                if (cell.possibleTiles.length < lowestEntropy) {
                    lowestEntropy = cell.possibleTiles.length;
                    lowestEntropyCells = [cell];
                } else if (cell.possibleTiles.length == lowestEntropy) {
                    lowestEntropyCells.push(cell);
                }
            }
        }
    }
    return lowestEntropyCells[Math.floor(Math.random() * lowestEntropyCells.length)];
}
function getRandomValueFromArray<T>(inputArray: T[]): T | undefined {
    if (inputArray.length === 0) {
        return undefined;
    }

    const randomIndex = Math.floor(Math.random() * inputArray.length);
    return inputArray[randomIndex];
}
function collapseStep() {
    var lowestEntropyCell = findLowestEntropy();
    lowestEntropyCell.path = getRandomValueFromArray(lowestEntropyCell.possibleTiles)!;
    lowestEntropyCell.possibleTiles = [lowestEntropyCell.path]

    var cellsNeedingPropagation = [lowestEntropyCell];
    var propogatedCells: Set<Cell> = new Set();

    const tilesMap = new Map(tiles.map(tile => [tile.path, tile]));

    while (cellsNeedingPropagation.length > 0) {
        var cell = cellsNeedingPropagation.pop()!;
        propogatedCells.add(cell);


        if (cell.col > 0) {
            var leftCell = board[cell.row][cell.col - 1];
            if (leftCell.path == "") {
                var valid: Set<string> = new Set();
                var set = cell.possibleTiles;
                for (let index = 0; index < set.length; index++) {
                    var t = tilesMap.get(set[index])!;
                    t.validLeft.forEach(value => valid.add(value));
                }
                leftCell.possibleTiles = leftCell.possibleTiles.filter(x => valid.has(x));

                if (!propogatedCells.has(leftCell)) {
                    cellsNeedingPropagation.push(leftCell);
                };
            }
        }
        if (cell.col < board[0].length - 1) {
            var rightCell = board[cell.row][cell.col + 1];
            if (rightCell.path == "") {
                var valid: Set<string> = new Set();
                var set = cell.possibleTiles;
                for (let index = 0; index < set.length; index++) {
                    var t = tilesMap.get(set[index])!;
                    t.validRight.forEach(value => valid.add(value));
                }
                rightCell.possibleTiles = rightCell.possibleTiles.filter(x => valid.has(x));

                if (!propogatedCells.has(rightCell)) {
                    cellsNeedingPropagation.push(rightCell);

                };
            }
        }

        if (cell.row > 0) {
            var topCell = board[cell.row - 1][cell.col];
            if (topCell.path == "") {
                var valid: Set<string> = new Set();
                var set = cell.possibleTiles;
                for (let index = 0; index < set.length; index++) {
                    var t = tilesMap.get(set[index])!;
                    t.validTop.forEach(value => valid.add(value));
                }
                topCell.possibleTiles = topCell.possibleTiles.filter(x => valid.has(x));

                if (!propogatedCells.has(topCell)) {
                    cellsNeedingPropagation.push(topCell);

                };
            }
        }
        if (cell.row < board.length - 1) {
            var bottomCell = board[cell.row + 1][cell.col];
            if (bottomCell.path == "") {
                var valid: Set<string> = new Set();
                var set = cell.possibleTiles;
                for (let index = 0; index < set.length; index++) {
                    var t = tilesMap.get(set[index])!;
                    t.validBottom.forEach(value => valid.add(value));
                }
                bottomCell.possibleTiles = bottomCell.possibleTiles.filter(x => valid.has(x));

                if (!propogatedCells.has(bottomCell)) {
                    cellsNeedingPropagation.push(bottomCell);

                };
            }
        }
        continue;
    }
}

async function renderBoard() {
    for (let x = 0; x < board.length; x++) {
        const row = board[x];
        for (let y = 0; y < row.length; y++) {
            const cell = row[y];
            if (cell.path == "" || drawnTiles.has(`${x},${y}`)) {
                continue;
            }
            drawnTiles.add(`${x},${y}`);

            if (!imageCache[cell.path]) {
                // If not in the cache, load the image and add it to the cache
                const image = new Image();
                image.src = "/tiles/custom_circuit/" + cell.path;
                await new Promise(resolve => image.onload = resolve);
                imageCache[cell.path] = image;
            }
            canvasctx!.drawImage(imageCache[cell.path], y * tileSize, x * tileSize, tileSize, tileSize)
            await new Promise(resolve => setTimeout(resolve, 1));
        }
    }
    gif.addFrame(canvasctx, { delay: 5000 / Math.floor(bh * bw), copy: true });
}



onMounted(async () => {
    canvasctx = canvas.value?.getContext("2d");

    const fileNames: string[] = [
        "empty.png",
        'circuit_0000s_0000_trace_c_l.png',
        'circuit_0000s_0001_trace_c_b.png',
        'circuit_0000s_0002_trace_c_r.png',
        'circuit_0000s_0003_trace_c_t.png',
        'circuit_0000s_0004_trace_t_l.png',
        'circuit_0000s_0005_trace_t_b.png',
        'circuit_0000s_0006_trace_t_r.png',
        'circuit_0000s_0007_trace_t_t.png',
        'circuit_0000s_0008_trace_h.png',
        'circuit_0000s_0009_trace_v.png',
        'circuit_0001s_0000_pad_circle_r.png',
        'circuit_0001s_0001_pad_circle_t.png',
        'circuit_0001s_0002_pad_circle_l.png',
        'circuit_0001s_0003_pad_circle_b.png',
        'circuit_0002s_0000_pad_rect_b.png',
        'circuit_0002s_0001_pad_rect_r.png',
        'circuit_0002s_0002_pad_rect_t.png',
        'circuit_0002s_0003_pad_rect_l.png',
        'circuit_0003s_0000_capacitor_l.png',
        'circuit_0003s_0001_capacitor_b.png',
        'circuit_0003s_0002_capacitor_r.png',
        'circuit_0003s_0003_capacitor_t.png',
        'circuit_0004s_0000s_0000_chip_2_corner_l.png',
        'circuit_0004s_0000s_0001_chip_2_corner_b.png',
        'circuit_0004s_0000s_0002_chip_2_corner_r.png',
        'circuit_0004s_0000s_0003_chip_2_corner_t.png',
        'circuit_0004s_0000_chip_2_full.png',
        'circuit_0004s_0001s_0000_chip_2_edge_connector_b.png',
        'circuit_0004s_0001s_0001_chip_2_edge_connector_t.png',
        'circuit_0004s_0001s_0002_chip_2_edge_connector_r.png',
        'circuit_0004s_0001s_0003_chip_2_edge_connector_l.png',
        'circuit_0004s_0001s_0004_chip_2_edge_b.png',
        'circuit_0004s_0001s_0005_chip_2_edge_t.png',
        'circuit_0004s_0001s_0006_chip_2_edge_r.png',
        'circuit_0004s_0001s_0007_chip_2_edge_l.png',
        'circuit_0005s_0000_chip_1_t.png',
        'circuit_0005s_0001_chip_1_l.png',
        'circuit_0005s_0002_chip_1_b.png',
        'circuit_0005s_0003_chip_1_r.png',
        'circuit_0005s_0004_chip_1.png',
        'circuit_0006s_0000_ground.png',
        'circuit_0006s_0001_ground_c_b.png',
        'circuit_0006s_0002_ground_c_l.png',
        'circuit_0006s_0003_ground_c_t.png',
        'circuit_0006s_0004_ground_c_r.png',
        'circuit_0007s_0000_resistor_2_r.png',
        'circuit_0007s_0001_resistor_2_t.png',
        'circuit_0008s_0000_resistor_1_r.png',
        'circuit_0008s_0001_resistor_1_t.png'
    ];

    tiles = await loadTiles("/tiles/custom_circuit/", fileNames, tileSize);
    generateValid();

    const validPaths: string[] = ['circuit_0004s_0000s_0000_chip_2_corner_l.png',
        'circuit_0004s_0000s_0001_chip_2_corner_b.png',
        'circuit_0004s_0000s_0002_chip_2_corner_r.png',
        'circuit_0004s_0000s_0003_chip_2_corner_t.png',
        'circuit_0004s_0000_chip_2_full.png',
        'circuit_0004s_0001s_0000_chip_2_edge_connector_b.png',
        'circuit_0004s_0001s_0001_chip_2_edge_connector_t.png',
        'circuit_0004s_0001s_0002_chip_2_edge_connector_r.png',
        'circuit_0004s_0001s_0003_chip_2_edge_connector_l.png',
        'circuit_0004s_0001s_0004_chip_2_edge_b.png',
        'circuit_0004s_0001s_0005_chip_2_edge_t.png',
        'circuit_0004s_0001s_0006_chip_2_edge_r.png',
        'circuit_0004s_0001s_0007_chip_2_edge_l.png'
    ]

    tiles.forEach(tile => {

        if (!validPaths.includes(tile.path)) {
            tile.validTop = tile.validTop.filter(x => x != tile.path)
            tile.validBottom = tile.validBottom.filter(x => x != tile.path)
            tile.validLeft = tile.validLeft.filter(x => x != tile.path)
            tile.validRight = tile.validRight.filter(x => x != tile.path)
        }
    });
    board = generateBoard(bh, bw);
    canvas.value!.height = bh * tileSize;
    canvas.value!.width = bw * tileSize;

    gif = new GIF({
        workers: 8,
        quality: 10,
        width: canvas.value!.width,
        height: canvas.value!.height,
        workerScript: '/js/gif.worker.js'
    });
})

function download() {
    var link = document.createElement('a');
    link.download = 'render.png';
    link.href = canvas.value!.toDataURL();
    link.click();
    link.remove();
}
function renderGIF() {
    gif.on('finished', function (blob: any) {
        window.open(URL.createObjectURL(blob));
    });
    gif.on('progress', function (p: any) {
        console.log(p);
    });

    gif.render();

}
</script>

<template>
    <canvas ref="canvas" width="0" height="0" style="border: solid 1px red;"></canvas>
    <button @click="stepsim(true);">Step</button>
    <button @click="autoPlay">Complete</button>
    <!--Save canvas as image and download-->
    <button @click="download()">Download as image</button>
    <button @click="renderGIF()">Render GIF</button>
</template>