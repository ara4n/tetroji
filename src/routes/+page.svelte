<script>
    /* TODO:
        * Start pieces in the right place (centered and off the top?)
        * Give the user more time if they're sliding or rotating to let them nudge the piece
        * Check kickers work correctly on collisions with existing shapes
        * Check for lines at the beginning of each turn rather than immediately
        * Animate line checks
        * Show drop shadow
        * Speed up as the game progresses
        * Keep score
        * check whether slicing the grid breaks svelte's speedups
     */
    let grid = [[]]; // [1,0] is top left
    let disappearing = {}; // which rows are disappearing

    const width = 10;
    const height = 24;

    // we draw the well (1 row off the bottom and 1 column to the left & right)
    for (let j = 0; j < height + 1; j++) {
        grid[j] = [];
        for (let i = 0; i < width + 2; i++) {
            grid[j][i] = (i == 0 || i == width + 1 || j == height) ? 8 : 0;
        }
    }

    // following https://tetris.wiki/Super_Rotation_System#How_Guideline_SRS_Really_Works
    const shapes = [ // [0, 0] is top left.
        { blocks: [[0, 0], [1, 0], [1, 1], [2, 1]], w: 3, h: 3 }, // Z
        { blocks: [[0, 1], [1, 1], [2, 1], [0, 0]], w: 3, h: 3 }, // J
        { blocks: [[0, 1], [1, 1], [2, 1], [2, 0]], w: 3, h: 3 }, // L
        { blocks: [[0, 0], [1, 0], [0, 1], [1, 1]], w: 2, h: 2 }, // O (simplified from SRS)
        { blocks: [[0, 1], [1, 1], [1, 0], [2, 0]], w: 3, h: 3 }, // S
        { blocks: [[0, 1], [1, 1], [2, 1], [1, 0]], w: 3, h: 3 }, // T
        { blocks: [[1, 2], [2, 2], [3, 2], [4, 2]], w: 5, h: 5 }, // I - SRS
    //  { blocks: [[0, 2], [1, 2], [2, 2], [3, 2]], w: 4, h: 4 }, // I (simplified from SRS)
    //  { blocks: [[1, 1], [2, 1], [1, 2], [2, 2]], w: 3, h: 3 }, // O - SRS
    ];

    // the offsets to test 3x3 blocks after N rotations
    const kickTests = [
        [[0, 0], [ 0, 0], [ 0, 0], [ 0, 0], [ 0, 0]], // 0
        [[0, 0], [+1, 0], [+1,-1], [ 0,+2], [+1,+2]], // R
        [[0, 0], [ 0, 0], [ 0, 0], [ 0, 0], [ 0, 0]], // 2
        [[0, 0], [-1, 0], [-1,-1], [ 0,+2], [-1,+2]], // L
    ];

    // the offsets to test for a 5x5 block (i.e. I bar)
    // compensates for the wobble of rotating within a 5x5
    const kickTestsI = [
        [[ 0, 0], [-1, 0], [+2, 0], [-1, 0], [+2, 0]], // 0
        [[-1, 0], [ 0, 0], [ 0, 0], [ 0,+1], [ 0,-2]], // R
        [[-1,+1], [+1,+1], [-2,+1], [+1, 0], [-2, 0]], // 2
        [[ 0,+1], [ 0,+1], [ 0,+1], [ 0,-1], [ 0,+2]], // L
    ];

    class Piece {
        type;
        x;
        y;
        rot;

        constructor() {
            this.type = Math.floor(Math.random() * 7) + 1;
            this.x = width / 2; // should be centered properly
            this.y = 0;
            this.rot = 0;
        }

        getShape() {
            return {
                blocks: shapes[this.type - 1].blocks.map((val)=>[...val]),
                w: shapes[this.type - 1].w,
                h: shapes[this.type - 1].h,
            };
        }

        getBlocks(x, y, rot) {
            let shape = this.getShape();
            //console.log(this.type, shape);
            const blocks = [];
            for (let i = 0; i < 4; i++) {
                blocks[i] = [];
                switch (rot) {
                    case 0:
                        blocks[i] = shape.blocks[i];
                        break;
                    case 1:
                        blocks[i][0] = shape.h - 1 - shape.blocks[i][1];
                        blocks[i][1] = shape.blocks[i][0];
                        break;
                    case 2:
                        blocks[i][0] = shape.h - 1 - shape.blocks[i][0];
                        blocks[i][1] = shape.w - 1 - shape.blocks[i][1];
                        break;
                    case 3:
                        blocks[i][0] = shape.blocks[i][1];
                        blocks[i][1] = shape.w - 1 - shape.blocks[i][0];
                        break;
                }
                blocks[i][0] += x;
                blocks[i][1] += y;
            }
            shape.blocks = blocks;
            return shape;
        }

        move(x, y, rot) {
            const oldBlocks = this.getBlocks(this.x, this.y, this.rot).blocks;

            //console.log("attempt move ", this, " to: ", x, y, rot);

            // clear old position
            for (let i = 0; i < 4; i++) {
                //console.log("clear ", oldBlocks[i]);
                grid[ oldBlocks[i][1] ][ oldBlocks[i][0] ] = 0;
            }

            const collision = () => {
                //console.log("testing piece ", this, " for: ", x, y, rot);
                const blocks = this.getBlocks(x, y, rot).blocks;
                for (let i = 0; i < 4; i++) {
                    if (grid[ blocks[i][1] ][ blocks[i][0] ] > 0) return true;
                }
            }

            let collided = false;
            if (rot != this.rot && this.type !== 4 /* O-piece */) {
                // check for wall kicks for this type of piece.
                const tests = this.type == 7 ? kickTestsI : kickTests;
                const offsetBefore = tests[this.rot];
                const offsetAfter = tests[rot];
                //console.log("kick offsets before & after: ", offsetBefore, offsetAfter);
                const oldX = x, oldY = y;
                for (let i = 0; i < offsetBefore.length; i++) {
                    //console.log(`trying to kick from rot ${this.rot} to ${rot}, offset: ${offsetAfter[i][0] - offsetBefore[i][0]},${offsetAfter[i][1] - offsetBefore[i][1]}`);
                    x = oldX - (offsetAfter[i][0] - offsetBefore[i][0]); // XXX: unsure why the offset has to be flipped here
                    y = oldY + (offsetAfter[i][1] - offsetBefore[i][1]);
                    collided = collision();
                    if (collided) {
                        x = oldX;
                        y = oldY;
                    }
                    else {
                        break;
                    }
                }
            }
            else {
                collided = collision();
            }

            if (collided) {
                console.log(`collision at ${x},${y}`);

                // reset old position
                for (let i = 0; i < 4; i++) {
                    //console.log("reset ", oldBlocks[i]);
                    grid[ oldBlocks[i][1] ][ oldBlocks[i][0] ] = this.type;
                }
                return false;
            }
            else {
                //console.log("moving piece to: ", x, y, rot);

                // draw new position
                const blocks = this.getBlocks(x, y, rot).blocks;
                for (let i = 0; i < 4; i++) {
                    //console.log("draw ", blocks[i]);
                    grid[ blocks[i][1] ][ blocks[i][0] ] = this.type;
                }

                this.x = x;
                this.y = y;
                this.rot = rot;

                // redraw the grid in svelte
                //grid = grid;

                return true;
            }
        }
    };

    let piece;
    let running = true;

    const setAsyncTimeout = (cb, timeout = 0) => new Promise(resolve => {
        setTimeout(() => {
            cb();
            resolve();
        }, timeout);
    });

    function removeLines() {
        const lines = Object.keys(disappearing).sort((a,b)=>(a-b));
        for (const line of lines) {
            for (let y = line; y > 0; y--) {
                for (let x = 1; x < width + 1; x++) {
                    grid[y][x] = grid[y - 1][x];
                }
            }
            for (let x = 1; x < width + 1; x++) {
                grid[0][x] = 0;
            }
        }
        console.log("Removed ", lines);
        disappearing = {};
    }

    async function checkForLines() {
        let doRemove = false;
        for (let j = height - 1; j >= 0; j--) {
            let blocks = 0;
            for (let i = 1; i < width + 1; i++) {
                blocks += grid[j][i] ? 1 : 0;
            }
            if (blocks == width && !disappearing[j]) {
                disappearing[j] = true; // set the animation running
                doRemove = true;
            }
        }
        if (doRemove) {
            console.log("removing lines", Object.keys(disappearing));
            await setAsyncTimeout(removeLines, 200);
        }
    }

    async function onKeyDown(e) {
        switch(e.keyCode) {
            case 38: // up arrow
                piece.move(piece.x, piece.y, (piece.rot + 1) % 4);
                break;
            case 40: // down arrow
                piece.move(piece.x, piece.y + 1, piece.rot);
                break;
            case 37: // left arrow
                piece.move(piece.x - 1, piece.y, piece.rot);
                break;
            case 39: // right arrow
                piece.move(piece.x + 1, piece.y, piece.rot);
                break;
            case 32: // space bar
                let success = false;
                do { success = piece.move(piece.x, piece.y + 1, piece.rot) } while (success);
                await checkAfterDrop();
                break;
         }
         return true;
    }

    async function checkAfterDrop() {
        await checkForLines();
        if (piece && piece.y == 0) running = false;
        piece = new Piece();
        console.log("started new piece: ", piece);
        piece.move(piece.x, piece.y, piece.rot);
    }

    async function runLoop() {
        // pause the loop while we're busy disappearing lines, otherwise we'll double-disappear them
        if (Object.keys(disappearing).length > 0) return;

        if (piece !== undefined) {
            const success = piece.move(piece.x, piece.y + 1, piece.rot);
            if (!success) {
                await checkAfterDrop();
            }
        }
        else {
            await checkAfterDrop();
        }
    }

    async function run() {
        await runLoop();
        while(running) {
            await setAsyncTimeout(runLoop, 1000);
        }
    }

    run();

</script>

<div class="tetroji">
    {#each grid.slice(0, height) as row, j}
        <div class="row { disappearing[j] ? 'disappearing' : ''}">
        {#each row.slice(1, width + 1) as cell}
            { cell ? String.fromCodePoint( 128996 + cell ) : '⬜' }
        {/each}
        </div>
    {/each}
</div>

<style>
    .tetroji {
        line-height: 1em;
    }

    .row {
        transition: opacity 0.2s;
    }

    .disappearing {
        opacity: 0;
    }
</style>

<svelte:window on:keydown={onKeyDown} />