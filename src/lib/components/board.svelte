<script lang="ts">
    import { onMount } from "svelte";

    enum Piece {
        Pawn,
        Knight,
        Bishop,
        Rook,
        Queen,
        King,
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

    const board: Square[][] = new Array();
    const startingBoardString = "♜♞♝♛♚♝♞♜♟♟♟♟♟♟♟♟8888♙♙♙♙♙♙♙♙♖♘♗♕♔♗♘♖";

    for (let i = 0; i < 8; i++) {
        board.push(new Array(8));
        for (let j = 0; j < 8; j++) {
            board[i][j] = {
                piece: null,
                color: (i + j) % 2 == 0 ? Color.Light : Color.Dark,
            };
        }
    }

    let highlight: {
        i: number | null;
        j: number | null;
    } = {
        i: null,
        j: null,
    };

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

    onMount(() => {
        squareSize = calculateSquareSize();

        window.addEventListener("resize", () => {
            squareSize = calculateSquareSize();
        });

        document.querySelectorAll(".piece").forEach((e) => {
            e.addEventListener("click", (e) => {
                const piece = e.target as HTMLElement;
                const i = parseInt(piece.classList[1].split("-")[1]);
                const j = parseInt(piece.classList[2].split("-")[1]);
                const pieceEntity = board[i][j].piece;
                if (!pieceEntity) return;
                highlight = {
                    i,
                    j,
                };
            });
        });
    });
</script>

<div class="self-center select-none">
    <div
        class="grid grid-cols-8 grid-rows-8 gap-0 rounded overflow-hidden shadow-md shadow-slate-950"
    >
        {#each board as row, i}
            {#each row as square, j}
                <div style="width: {squareSize}px; height: {squareSize}px;">
                    <div
                        class="w-full h-full {square.color === Color.Light
                            ? i !== highlight.i || j !== highlight.j
                                ? 'bg-slate-300'
                                : 'bg-green-300'
                            : i !== highlight.i || j !== highlight.j
                            ? 'bg-slate-400'
                            : 'bg-green-400'} "
                    >
                        {#if square.piece}
                            <div
                                style="font-size: {squareSize * 0.7}px; 
                                    line-height: {squareSize}px;"
                                class="piece i-{i} j-{j} text-center"
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
