<script setup lang="ts">
import GIF from 'gif.js';

var canvas = ref<HTMLCanvasElement>();
var canvasctx = canvas.value?.getContext("2d");
var board: Cell[][];
var stepsMade = 0;
var tiles: Tile[];
var tileSize = 14;
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
                image.src = "/tiles/Circuit/" + cell.path;
                await new Promise(resolve => image.onload = resolve);
                imageCache[cell.path] = image;
            }
            canvasctx!.drawImage(imageCache[cell.path], y * tileSize, x * tileSize, tileSize, tileSize)
            await new Promise(resolve => setTimeout(resolve, 1));
        }
    }
    gif.addFrame(canvasctx, { delay: 5000/Math.floor(bh * bw), copy: true });
}



onMounted(async () => {
    canvasctx = canvas.value?.getContext("2d");

    const fileNames: string[] = [
        "bridge.png",
        "bridge_0.png",
        "component.png",
        "connection.png",
        "connection_0.png",
        "connection_1.png",
        "connection_2.png",
        "corner.png",
        "corner_0.png",
        "corner_1.png",
        "corner_2.png",
        "skew.png",
        "skew_0.png",
        "skew_1.png",
        "skew_2.png",
        "substrate.png",
        "t.png",
        "track.png",
        "track_1.png",
        "transition.png",
        "transition_0.png",
        "transition_1.png",
        "transition_2.png",
        "turn.png",
        "turn_0.png",
        "turn_1.png",
        "turn_2.png",
        "t_0.png",
        "t_1.png",
        "t_2.png",
        "viad.png",
        "viad_0.png",
        "vias.png",
        "vias_0.png",
        "vias_1.png",
        "vias_2.png",
        "wire.png",
        "wire_0.png",
    ];

    tiles = await loadTiles("/tiles/Circuit/", fileNames, tileSize);
    generateValid();

    tiles[tiles.findIndex(x => x.path == "corner.png")].validBottom = tiles[tiles.findIndex(x => x.path == "corner.png")].validBottom.filter(x => x != "corner_0.png")
    tiles[tiles.findIndex(x => x.path == "corner_0.png")].validLeft = tiles[tiles.findIndex(x => x.path == "corner_0.png")].validLeft.filter(x => x != "corner_1.png")
    tiles[tiles.findIndex(x => x.path == "corner_1.png")].validTop = tiles[tiles.findIndex(x => x.path == "corner_1.png")].validTop.filter(x => x != "corner_2.png")
    tiles[tiles.findIndex(x => x.path == "corner_2.png")].validRight = tiles[tiles.findIndex(x => x.path == "corner_2.png")].validRight.filter(x => x != "corner.png")

    tiles[tiles.findIndex(x => x.path == "corner_0.png")].validTop = tiles[tiles.findIndex(x => x.path == "corner_0.png")].validTop.filter(x => x != "corner.png")
    tiles[tiles.findIndex(x => x.path == "corner_1.png")].validRight = tiles[tiles.findIndex(x => x.path == "corner_1.png")].validRight.filter(x => x != "corner_0.png")
    tiles[tiles.findIndex(x => x.path == "corner_2.png")].validBottom = tiles[tiles.findIndex(x => x.path == "corner_2.png")].validBottom.filter(x => x != "corner_1.png")
    tiles[tiles.findIndex(x => x.path == "corner.png")].validLeft = tiles[tiles.findIndex(x => x.path == "corner.png")].validLeft.filter(x => x != "corner_2.png")

    //tiles[tiles.findIndex(x => x.path == "skew.png")].validTop = tiles[tiles.findIndex(x => x.path == "skew.png")].validTop.filter(x => x != "skew_0.png")
    //tiles[tiles.findIndex(x => x.path == "skew_0.png")].validRight = tiles[tiles.findIndex(x => x.path == "skew_0.png")].validRight.filter(x => x != "skew_1.png")
    //tiles[tiles.findIndex(x => x.path == "skew_1.png")].validBottom = tiles[tiles.findIndex(x => x.path == "skew_1.png")].validBottom.filter(x => x != "skew_2.png")
    //tiles[tiles.findIndex(x => x.path == "skew_2.png")].validLeft = tiles[tiles.findIndex(x => x.path == "skew_2.png")].validLeft.filter(x => x != "skew.png")

    //tiles[tiles.findIndex(x => x.path == "skew_0.png")].validBottom = tiles[tiles.findIndex(x => x.path == "skew_0.png")].validBottom.filter(x => x != "skew.png")
    //tiles[tiles.findIndex(x => x.path == "skew_1.png")].validLeft = tiles[tiles.findIndex(x => x.path == "skew_1.png")].validLeft.filter(x => x != "skew_0.png")
    //tiles[tiles.findIndex(x => x.path == "skew_2.png")].validTop = tiles[tiles.findIndex(x => x.path == "skew_2.png")].validTop.filter(x => x != "skew_1.png")
    //tiles[tiles.findIndex(x => x.path == "skew.png")].validRight = tiles[tiles.findIndex(x => x.path == "skew.png")].validRight.filter(x => x != "skew_2.png")

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