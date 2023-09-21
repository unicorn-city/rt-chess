<script lang="ts">
    export let mute = false;

    import { onMount } from "svelte";

    enum Piece {
        Pawn = "Pawn",
        Knight = "Knight",
        Bishop = "Bishop",
        Rook = "Rook",
        Queen = "Queen",
        King = "King",
    }

    enum Color {
        Light,
        Dark,
    }

    type PieceEntity = {
        piece: Piece;
        color: Color;
    };

    type Square = {
        piece: PieceEntity | null;
        color: Color;
        highlight: boolean;
    };

    const pieceToGylphDict: Record<Color, Record<Piece, string>> = {
        [Color.Dark]: {
            [Piece.Pawn]: "♟",
            [Piece.Knight]: "♞",
            [Piece.Bishop]: "♝",
            [Piece.Rook]: "♜",
            [Piece.Queen]: "♛",
            [Piece.King]: "♚",
        },
        [Color.Light]: {
            [Piece.Pawn]: "♙",
            [Piece.Knight]: "♘",
            [Piece.Bishop]: "♗",
            [Piece.Rook]: "♖",
            [Piece.Queen]: "♕",
            [Piece.King]: "♔",
        },
    };

    const glyphToPieceDict: Record<string, PieceEntity> = {
        "♟": {
            piece: Piece.Pawn,
            color: Color.Dark,
        },
        "♞": {
            piece: Piece.Knight,
            color: Color.Dark,
        },
        "♝": {
            piece: Piece.Bishop,
            color: Color.Dark,
        },
        "♜": {
            piece: Piece.Rook,
            color: Color.Dark,
        },
        "♛": {
            piece: Piece.Queen,
            color: Color.Dark,
        },
        "♚": {
            piece: Piece.King,
            color: Color.Dark,
        },

        "♙": {
            piece: Piece.Pawn,
            color: Color.Light,
        },
        "♘": {
            piece: Piece.Knight,
            color: Color.Light,
        },
        "♗": {
            piece: Piece.Bishop,
            color: Color.Light,
        },
        "♖": {
            piece: Piece.Rook,
            color: Color.Light,
        },
        "♕": {
            piece: Piece.Queen,
            color: Color.Light,
        },
        "♔": {
            piece: Piece.King,
            color: Color.Light,
        },
    };

    type Coordinates = {
        i: number;
        j: number;
    };

    const board: Square[][] = new Array();
    const startingBoardString = "♜♞♝♛♚♝♞♜♟♟♟♟♟♟♟♟8888♙♙♙♙♙♙♙♙♖♘♗♕♔♗♘♖";

    for (let i = 0; i < 8; i++) {
        board.push(new Array(8));
        for (let j = 0; j < 8; j++) {
            board[i][j] = {
                piece: null,
                color: (i + j) % 2 == 0 ? Color.Light : Color.Dark,
                highlight: false,
            };
        }
    }

    let highlight: Coordinates | null = null;

    {
        let squareIndex = 0;
        for (const c of startingBoardString) {
            const cNumber = parseInt(c);
            if (!isNaN(cNumber)) {
                squareIndex += cNumber;
                continue;
            }
            board[Math.floor(squareIndex / 8)][squareIndex % 8].piece = {
                ...glyphToPieceDict[c],
                color: squareIndex < 32 ? Color.Dark : Color.Light,
            };
            squareIndex++;
        }
    }

    let squareSize = 12;
    const calculateSquareSize = () =>
        (squareSize = Math.min(
            64,
            (Math.min(window.innerHeight, window.innerWidth) - 64) / 8
        ));

    let selectedPiece: (PieceEntity & Coordinates) | null = null;
    let pieceHeld = false;

    let mousePosition = { x: 0, y: 0 };

    let moveAudio: HTMLAudioElement;
    let takeAudio: HTMLAudioElement;

    const calculateSquareCoordinates = (e: TouchEvent): Coordinates => {
        const touchX = e.changedTouches[0].clientX;
        const touchY = e.changedTouches[0].clientY;

        // couldn't abstract this, board size changes extra
        const boardRect = document
            .querySelector(".board")
            ?.getBoundingClientRect();

        if (!boardRect) throw new Error("boardRect is null");
        const i = Math.floor((touchY - boardRect.top) / squareSize);
        const j = Math.floor((touchX - boardRect.left) / squareSize);

        return { i, j };
    };

    const handlePressDown = (
        e: MouseEvent | TouchEvent,
        i: number,
        j: number
    ) => {
        const coord = { i, j };
        const pieceEntity = board[i][j].piece;

        // if there is a selected piece move it now
        if (selectedPiece && (selectedPiece.i !== i || selectedPiece.j !== j)) {
            if (!mute)
                if (board[i][j].piece) {
                    takeAudio.currentTime = 0;
                    takeAudio.play();
                } else {
                    moveAudio.currentTime = 0;
                    moveAudio.play();
                }

            board[i][j].piece = selectedPiece;
            board[selectedPiece.i][selectedPiece.j].piece = null;
            selectedPiece = null;
            pieceHeld = false;
            highlight = null;
            // else select it and prepare to move it
        } else {
            // TODO: why does tap-tap (not drag-release) reselect placed piece for touch? 
            if (!pieceEntity) return;

            selectedPiece = { ...pieceEntity, ...coord };

            mousePosition =
                e instanceof MouseEvent
                    ? { x: e.clientX, y: e.clientY }
                    : { x: e.touches[0].clientX, y: e.touches[0].clientY };

            pieceHeld = true;
            highlight = coord;

            if (e instanceof TouchEvent) {
                e.target?.addEventListener("touchmove", handleTouchMove);

                const handleTouchEvent = (ee: Event) => {
                    const touchEvent = ee as TouchEvent;
                    const { i, j } = calculateSquareCoordinates(touchEvent);
                    handlePressUp(touchEvent, i, j);
                    e.target?.removeEventListener("touchmove", handleTouchMove);
                };
                e.target?.addEventListener("touchend", handleTouchEvent);
                e.target?.addEventListener("touchcancel", handleTouchEvent);
            }
        }
    };

    const handlePressUp = (
        e: MouseEvent | TouchEvent,
        i: number,
        j: number
    ) => {
        e.preventDefault();
        if (!pieceHeld || !selectedPiece) return;
        if (selectedPiece.i === i && selectedPiece.j === j) {
            pieceHeld = false;
            return;
        } else {
            if (!mute)
                if (board[i][j].piece) {
                    takeAudio.currentTime = 0;
                    takeAudio.play();
                } else {
                    moveAudio.currentTime = 0;
                    moveAudio.play();
                }

            board[i][j].piece = selectedPiece;
            board[selectedPiece.i][selectedPiece.j].piece = null;
            selectedPiece = null;
            pieceHeld = false;
            highlight = null;
        }
    };

    function handleMouseMove(e) {
        e.preventDefault();
        mousePosition = { x: e.clientX, y: e.clientY };
    }

    function handleTouchMove(e) {
        e.preventDefault();
        let touch = e.touches[0];
        if (touch) mousePosition = { x: touch.clientX, y: touch.clientY };
    }

    onMount(() => {
        moveAudio = new Audio("./sounds/bass_boop.mp3");
        moveAudio.volume = 0.2;
        takeAudio = new Audio("./sounds/killshot.mp3");
        takeAudio.volume = 0.5;

        squareSize = calculateSquareSize();
    });

    $: displayPiece =
        selectedPiece && pieceHeld
            ? pieceToGylphDict[selectedPiece.color][selectedPiece.piece]
            : "";
</script>

<svelte:window
    on:resize={() => {
        squareSize = calculateSquareSize();
    }}
/>

<div class="self-center select-none touch-none">
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <div
        class="board grid grid-cols-8 grid-rows-8 gap-0 rounded overflow-hidden shadow-md shadow-slate-950 touch-none"
        on:mousemove={handleMouseMove}
    >
        <div
            style="
                font-size: {squareSize * 0.7}px; 
                line-height: {squareSize}px; 
                width: {squareSize}px; 
                height: {squareSize}px;
                left: {mousePosition.x - squareSize / 2}px;
                top: {mousePosition.y - squareSize / 2}px;
                "
            class="absolute pointer-events-none text-center"
        >
            {displayPiece}
        </div>
        {#each board as row, i}
            {#each row as square, j}
                <div style="width: {squareSize}px; height: {squareSize}px;">
                    <!-- svelte-ignore a11y-no-noninteractive-tabindex -->
                    <div
                        on:mousedown={(e) => handlePressDown(e, i, j)}
                        on:touchstart={(e) => handlePressDown(e, i, j)}
                        on:mouseup={(e) => handlePressUp(e, i, j)}
                        tabindex={i * 8 + j}
                        class="w-full h-full {square.color === Color.Light
                            ? i !== highlight?.i || j !== highlight?.j
                                ? 'bg-slate-300'
                                : 'bg-green-300'
                            : i !== highlight?.i || j !== highlight?.j
                            ? 'bg-slate-400'
                            : 'bg-green-400'} "
                    >
                        {#if square.piece && !(pieceHeld && selectedPiece?.i === i && selectedPiece?.j === j)}
                            <div
                                style="
                                    font-size: {squareSize * 0.7}px; 
                                    line-height: {squareSize}px;
                                "
                                class="text-center"
                            >
                                {pieceToGylphDict[square.piece.color][
                                    square.piece.piece
                                ]}
                            </div>
                        {/if}
                    </div>
                </div>
            {/each}
        {/each}
    </div>
</div>
