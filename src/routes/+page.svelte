<script>
	export let grid = [[]];

	const width = 10;
	const height = 24;

	for (let j = 0; j < height + 1; j++) {
		grid[j] = [];
		for (let i = 0; i < width + 2; i++) {
			grid[j][i] = (i == 0 || i == width + 1 || j == height) ? 8 : 0;
		}
	}

	const shapes = [
		{ blocks: [[0, 1], [1, 1], [2, 1], [3, 1]], w: 4, h: 4 }, // bar
		{ blocks: [[0, 0], [1, 0], [0, 1], [1, 1]], w: 2, h: 2 }, // square
		{ blocks: [[0, 0], [1, 0], [2, 0], [1, 1]], w: 3, h: 3 }, // tee
		{ blocks: [[0, 0], [1, 0], [1, 1], [2, 1]], w: 3, h: 3 }, // s-left
		{ blocks: [[0, 1], [1, 1], [1, 0], [2, 0]], w: 3, h: 3 }, // s-right
		{ blocks: [[0, 1], [0, 0], [1, 0], [2, 0]], w: 3, h: 3 }, // l-left
		{ blocks: [[0, 0], [1, 0], [2, 0], [2, 1]], w: 3, h: 3 }, // r-right
	];

	class Piece {
		type;
		x;
		y;
		rot;

		constructor() {
			this.type = Math.floor(Math.random() * 7) + 1;
			this.x = width / 2;
			this.y = 0;
			this.rot = 0;
		}

		getShape() {
			return {
				blocks: shapes[this.type - 1].blocks.map((val)=>[...val]),
				w: shapes[this.type - 1].w,
				w: shapes[this.type - 1].h,
			};
		}

		getBlocks(x, y, rot) {
			let shape = this.getShape();
			console.log(this.type, shape);
			for (let i = 0; i < 4; i++) {
				switch (rot) {
					case 0:
						break;
					case 1:
						shape.blocks[i][0] = shape.h - 1 - shape[i][1];
						shape.blocks[i][1] = shape[i][0];
						break;
					case 2:
						shape[i][0] = shape.h - 1 - shape[i][0];
						shape[i][1] = shape.w - 1 - shape[i][1];
						break;
					case 3:
						shape[i][0] = shape[i][1];
						shape[i][1] = shape.w - 1 - shape[i][0];
						break;
				}
				shape[i][0] += x;
				shape[i][1] += y;
			}
			return shape;
		}

		move(x, y, rot) {
			const oldBlocks = this.getBlocks(this.x, this.y, this.rot);
			const blocks = this.getBlocks(x, y, rot);

			// clear old position
			for (let i = 0; i < 4; i++) {
				grid[ oldBlocks[i][1] ][ oldBlocks[i][0] ] = 0;
			}

			// test for collisions
			let collision = () => {
				console.log("testing piece for: ", x, y, rot);
				if (y > height - 4) return true;
				// TODO: crop to shape
				if (x < 0) return true;
				if (x > width - 4) return true;
				for (let i = 0; i < 4; i++) {
					if (grid[ blocks[i][1] ][ blocks[i][0] ] > 0) return true;
				}
			}

			// TODO: if rotating, push away from collisions

			if (collision()) {
				console.log("collision!");

				// reset old position
				for (let i = 0; i < 4; i++) {
					grid[ oldBlocks[i][1] ][ oldBlocks[i][0] ] = this.type;
				}
				return false;
			}
			else {
				console.log("moving piece to: ", x, y, rot);

				// draw new position
				for (let i = 0; i < 4; i++) {
					grid[ blocks[i][1] ][ blocks[i][0] ] = this.type;
				}

				piece.x = x;
				piece.y = y;
				piece.rot = rot;

				// redraw the grid in svelte
				grid = grid;

				return true;
			}
		}
	};

	let piece;
	let running = true;

	setInterval(() => {
		if (!running) return;

		// game loop
		if (piece === undefined) {
			piece = new Piece();
			console.log("started new piece: ", piece);
		} else {
			const success = piece.move(piece.x, piece.y + 1, piece.rot);
			if (!success) {
				if (piece.y == 0) running = false;
				piece = undefined;
			}
		}
	}, 100);

</script>

<table>
{#each grid as row}
	<tr>
	{#each row as cell}
		<td>
			{ cell ? String.fromCodePoint( 128996 + cell ) : '&nbsp;' }
		</td>
	{/each}
	</tr>
{/each}
</table>

