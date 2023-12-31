<script lang="ts">
	import { onMount } from "svelte";

	const PieceTypes = [
		{
			// Cube
			color: "orange",
			blocks: [
				{ x: 0, y: 0 },
				{ x: 1, y: 0 },
				{ x: 0, y: 1 },
				{ x: 1, y: 1 },
			],
		},
		{
			// Right L
			color: "purple",
			blocks: [
				{ x: 0, y: 0 },
				{ x: 1, y: 0 },
				{ x: 2, y: 0 },
				{ x: 2, y: 1 },
			],
		},
		{
			// Left L
			color: "pink",
			blocks: [
				{ x: 0, y: 1 },
				{ x: 1, y: 1 },
				{ x: 2, y: 1 },
				{ x: 2, y: 0 },
			],
		},
		{
			// I
			color: "yellow",
			blocks: [
				{ x: 0, y: 0 },
				{ x: 0, y: 1 },
				{ x: 0, y: 2 },
				{ x: 0, y: 3 },
			],
		},
		{
			// Zig 1
			color: "cyan",
			blocks: [
				{ x: 0, y: 0 },
				{ x: 1, y: 0 },
				{ x: 1, y: 1 },
				{ x: 2, y: 1 },
			],
		},
		{
			// Zig 2
			color: "red",
			blocks: [
				{ x: 1, y: 0 },
				{ x: 2, y: 0 },
				{ x: 0, y: 1 },
				{ x: 1, y: 1 },
			],
		},
		{
			// Triangle
			color: "green",
			blocks: [
				{ x: 1, y: 0 },
				{ x: 0, y: 1 },
				{ x: 1, y: 1 },
				{ x: 2, y: 1 },
			],
		},
	];
	const pieceAmount = PieceTypes.length;

	const BlockSize = 32;

	const gridWidth = 10;
	const gridHeight = 16;

	const tableWidth = BlockSize * gridWidth;
	const tableHeight = BlockSize * gridHeight;

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;

	let piecesToSelect: number[] = [];

	let currentPiece = createPiece(5);

	// 2D array
	let placedBlockRows: (string | undefined)[][] = new Array(gridHeight)
		.fill([])
		.map(() => new Array(gridWidth).fill(undefined));

	let lostGame = false;

	function copyPieceBlocks(piece: number) {
		let blocks: { x: number; y: number }[] = [];

		for (const block of PieceTypes[piece].blocks) {
			blocks.push({
				x: block.x,
				y: block.y,
			});
		}

		return blocks;
	}

	function getBlocksRow(row: number) {
		if (row < 0 || row > gridHeight) {
			return undefined;
		}

		return placedBlockRows[row];
	}

	function getBlock(x: number, y: number) {
		if (x < 0 || x > gridWidth) {
			return undefined;
		}

		let row = getBlocksRow(y);

		if (row) return row[x];
	}

	function setBlock(x: number, y: number, color: string) {
		let row = getBlocksRow(y);

		if (row) row[x] = color;
	}

	function clearRow(row: number) {
		placedBlockRows = placedBlockRows.filter((_, i) => i != row);
		// add a new row to the top
		placedBlockRows.unshift(new Array(gridWidth).fill(undefined));
	}

	function drawInGrid(x: number, y: number) {
		ctx.fillRect(x * BlockSize, y * BlockSize, BlockSize, BlockSize);
	}

	function getAllBlocks() {
		let blocks = [];

		let y = 0;
		for (const row of placedBlockRows) {
			let x = 0;
			for (const color of row) {
				if (color) blocks.push({ x, y, color });
				x++;
			}
			y++;
		}

		return blocks;
	}

	function clearCompletedRows() {
		let rowBlockAmounts = new Array<number>(gridHeight).fill(0);

		for (const block of getAllBlocks()) {
			rowBlockAmounts[block.y] += 1;
		}

		let rowsToClear: number[] = [];

		let i = 0;
		for (const rowBlockAmount of rowBlockAmounts) {
			if (rowBlockAmount == gridWidth) {
				rowsToClear.push(i);
			}
			i++;
		}

		for (const row of rowsToClear) {
			clearRow(row);
		}
	}

	function nextPiece() {
		if (piecesToSelect.length == 0) {
			piecesToSelect = new Array(pieceAmount).fill(0).map((_, i) => i);
		}

		let type =
			piecesToSelect[Math.floor(Math.random() * piecesToSelect.length)];

		piecesToSelect = piecesToSelect.filter((piece) => piece != type);

		currentPiece = createPiece(type);
	}

	function endGame() {
		lostGame = true;
	}

	function createPiece(type: number) {
		let blocks = copyPieceBlocks(type);

		let pos = {
			x: 0,
			y: 0,
		};

		function getLowestBlock() {
			let lowestBlock = blocks[0];

			for (const block of blocks) {
				if (block.y < lowestBlock.y) {
					lowestBlock = block;
				}
			}

			return lowestBlock;
		}

		function getHighestBlock() {
			let highestBlock = blocks[0];

			for (const block of blocks) {
				if (block.y > highestBlock.y) {
					highestBlock = block;
				}
			}

			return highestBlock;
		}

		function getHeight() {
			return getHighestBlock().y - getLowestBlock().y + 1;
		}

		function getLeftAndRightMostBlocks() {
			let leftmostBlock = blocks[0];
			let rightmostBlock = blocks[0];

			for (const block of blocks) {
				if (block.x > leftmostBlock.x) {
					leftmostBlock = block;
				}
				if (block.x < rightmostBlock.x) {
					rightmostBlock = block;
				}
			}

			return {
				leftmostBlock,
				rightmostBlock,
			};
		}

		function getWidth() {
			let { leftmostBlock, rightmostBlock } = getLeftAndRightMostBlocks();

			return leftmostBlock.x - rightmostBlock.x + 1;
		}

		function isOnFloor() {
			return pos.y + getLowestBlock().y >= gridHeight - getHeight();
		}

		function wouldCollideWithBlocks(
			blocks: {
				x: number;
				y: number;
			}[]
		) {
			for (const block of blocks) {
				if (getBlock(pos.x + block.x, pos.y + block.y) != undefined)
					return true;
			}

			return false;
		}

		function wouldCollideIfMovedTo(x: number, y: number) {
			return wouldCollideWithBlocks(
				blocks.map((block) => ({
					x: block.x + x,
					y: block.y + y,
				}))
			);
		}

		function hasBlockBelow() {
			return wouldCollideIfMovedTo(0, 1);
		}

		function isGameOver() {
			let row = placedBlockRows[0];

			for (const block of row) {
				if (block != undefined) {
					return true;
				}
			}
		}

		// returns how much of out bounds the piece is and in what direction
		function getOutOfBounds() {
			let { leftmostBlock, rightmostBlock } = getLeftAndRightMostBlocks();

			let stuff = -leftmostBlock.x - 1 + getWidth();

			if (pos.x < stuff) {
				return pos.x - stuff;
			}

			let stuff2 = gridWidth - (rightmostBlock.x + getWidth());

			if (pos.x > stuff2) {
				return pos.x - stuff2;
			}

			return 0;
		}

		function goBackIfOutOfBounds() {
			let outOfBounds = getOutOfBounds();

			if (outOfBounds) {
				pos.x -= outOfBounds;
			}
		}

		return {
			draw() {
				ctx.fillStyle = PieceTypes[type].color;

				for (const { x, y } of blocks) {
					drawInGrid(pos.x + x, pos.y + y);
				}
			},
			moveRight() {
				if (wouldCollideIfMovedTo(1, 0)) {
					goBackIfOutOfBounds();
					return;
				}

				pos.x += 1;
				goBackIfOutOfBounds();
			},
			moveLeft() {
				if (wouldCollideIfMovedTo(-1, 0)) {
					goBackIfOutOfBounds();
					return;
				}

				pos.x -= 1;
				goBackIfOutOfBounds();
			},
			moveDown() {
				if (hasBlockBelow() || isOnFloor()) {
					for (const block of blocks) {
						setBlock(pos.x + block.x, pos.y + block.y, PieceTypes[type].color);
					}

					if (isGameOver()) return endGame();

					nextPiece();
					clearCompletedRows();
				} else {
					pos.y += 1;
				}
			},
			rotate() {
				let newBlocks = [];

				for (const block of blocks) {
					newBlocks.push({
						x: block.y,
						y: -block.x,
					});
				}

				if (wouldCollideWithBlocks(newBlocks)) {
					return;
				}

				blocks = newBlocks;

				goBackIfOutOfBounds();
			},
		};
	}

	function mainloop() {
		// Clear screen
		ctx.fillStyle = "#131516";
		ctx.fillRect(0, 0, tableWidth, tableHeight);

		currentPiece.draw();

		for (const { x, y, color } of getAllBlocks()) {
			ctx.fillStyle = color;
			drawInGrid(x, y);
		}

		return requestAnimationFrame(mainloop);
	}

	function everySecond() {
		currentPiece.moveDown();
	}

	onMount(() => {
		ctx = canvas.getContext("2d") as CanvasRenderingContext2D;

		let secondFrame = setInterval(everySecond, 1000);
		let mainloopFrame = mainloop();

		return () => {
			cancelAnimationFrame(mainloopFrame);
			clearInterval(secondFrame);
		};
	});
</script>

<svelte:window
	on:keydown={(e) => {
		if (e.key == "ArrowRight") currentPiece.moveRight();
		else if (e.key == "ArrowLeft") currentPiece.moveLeft();
		else if (e.key == "ArrowDown") currentPiece.moveDown();
		else if (e.key == "ArrowUp") currentPiece.rotate();
	}}
/>

{#if lostGame}
	<main class="lost-game">
		<h1>You lost</h1>
		<button on:click={() => window.location.reload()}> Play again </button>
	</main>
{:else}
	<main class="game">
		<canvas bind:this={canvas} width={tableWidth} height={tableHeight} />
		<div class="buttons" style="width: {tableWidth}px;">
			<button on:click={currentPiece.moveLeft}> &larr; </button>
			<div class="buttons-2">
				<button on:click={currentPiece.rotate}> &#8635; </button>
				<button on:click={currentPiece.moveDown}> &darr; </button>
			</div>
			<button on:click={currentPiece.moveRight}> &rarr; </button>
		</div>
	</main>
{/if}

<style>
	:global(body) {
		margin: 0;
	}
	.lost-game {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		height: 100vh;
		font-size: xx-large;
	}
	.lost-game button {
		font-size: xx-large;
	}
	.game {
		display: flex;
		flex-direction: column;
		align-items: center;
		height: 100dvh;
	}
	canvas {
		height: 80%;
	}
	.buttons {
		display: flex;
		height: 20%;
	}
	.buttons-2 {
		display: flex;
		flex-direction: column;
		height: 100%;
		width: 100%;
	}
	.game button {
		font-size: xx-large;
		height: 100%;
		width: 100%;
	}
</style>
