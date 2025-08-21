import React, { useEffect, useMemo, useState, useRef } from "react";

// MinesGame // Single-file React component (Tailwind CSS assumed). // Default export is the component to drop into any React app (Vite / Create React App / Next.js). // Features: // - Playable Minesweeper-style game // - Difficulty selector // - AI/Bot that plays using basic deterministic rules + random fallback // - Lightweight, no external libs required (Tailwind for styling)

// Helper utilities const range = (n) => Array.from({ length: n }, (_, i) => i);

function generateBoard(rows, cols, mines, firstClick = null) { // create empty board const board = range(rows).map(() => range(cols).map(() => ({ mine: false, revealed: false, flagged: false, adjacent: 0, })));

// list of all positions except firstClick and its neighbors const forbidden = new Set(); if (firstClick) { const [r0, c0] = firstClick; for (let dr = -1; dr <= 1; dr++) { for (let dc = -1; dc <= 1; dc++) { const r = r0 + dr; const c = c0 + dc; if (r >= 0 && r < rows && c >= 0 && c < cols) forbidden.add(${r},${c}); } } }

const candidates = []; for (let r = 0; r < rows; r++) { for (let c = 0; c < cols; c++) { if (!forbidden.has(${r},${c})) candidates.push([r, c]); } }

// shuffle and place mines for (let i = candidates.length - 1; i > 0 && mines > 0; i--) { const j = Math.floor(Math.random() * (i + 1)); [candidates[i], candidates[j]] = [candidates[j], candidates[i]]; }

for (let i = 0; i < mines && i < candidates.length; i++) { const [r, c] = candidates[i]; board[r][c].mine = true; }

// compute adjacent counts for (let r = 0; r < rows; r++) { for (let c = 0; c < cols; c++) { if (board[r][c].mine) continue; let count = 0; for (let dr = -1; dr <= 1; dr++) { for (let dc = -1; dc <= 1; dc++) { if (dr === 0 && dc === 0) continue; const nr = r + dr; const nc = c + dc; if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && board[nr][nc].mine) count++; } } board[r][c].adjacent = count; } }

return board; }

function cloneBoard(board) { return board.map((row) => row.map((cell) => ({ ...cell }))); }

export default function MinesGame() { const presets = useMemo( () => ({ Beginner: { rows: 9, cols: 9, mines: 10 }, Intermediate: { rows: 16, cols: 16, mines: 40 }, Expert: { rows: 16, cols: 30, mines: 99 }, }), [] );

const [rows, setRows] = useState(presets.Beginner.rows); const [cols, setCols] = useState(presets.Beginner.cols); const [mines, setMines] = useState(presets.Beginner.mines); const [board, setBoard] = useState(() => generateBoard(rows, cols, mines)); const [started, setStarted] = useState(false); const [lost, setLost] = useState(false); const [won, setWon] = useState(false); const [flagsLeft, setFlagsLeft] = useState(mines); const [startTime, setStartTime] = useState(null); const [elapsed, setElapsed] = useState(0); const timerRef = useRef(null);

const [aiRunning, setAiRunning] = useState(false); const aiIntervalRef = useRef(null);

// Reset / new game function newGame(r = rows, c = cols, m = mines) { setStarted(false); setLost(false); setWon(false); setFlagsLeft(m); setStartTime(null); setElapsed(0); if (timerRef.current) { clearInterval(timerRef.current); timerRef.current = null; } if (aiIntervalRef.current) { clearInterval(aiIntervalRef.current); aiIntervalRef.current = null; } setAiRunning(false); setBoard(generateBoard(r, c, m)); }

// start timer when game starts useEffect(() => { if (started && !timerRef.current && !lost && !won) { setStartTime(Date.now()); timerRef.current = setInterval(() => { setElapsed(Math.floor((Date.now() - (startTime || Date.now())) / 1000)); }, 1000); } if ((lost || won) && timerRef.current) { clearInterval(timerRef.current); timerRef.current = null; } return () => {}; // eslint-disable-next-line react-hooks/exhaustive-deps }, [started, lost, won, startTime]);

// update flagsLeft when mines change useEffect(() => { setFlagsLeft(mines); newGame(rows, cols, mines); // eslint-disable-next-line react-hooks/exhaustive-deps }, [mines]);

// helper: reveal flood fill function reveal(cellBoard, r, c) { const rowsL = cellBoard.length; const colsL = cellBoard[0].length; const stack = [[r, c]]; const visited = new Set();

while (stack.length) {
  const [rr, cc] = stack.pop();
  const key = `${rr},${cc}`;
  if (visited.has(key)) continue;
  visited.add(key);
  const cell = cellBoard[rr][cc];
  if (cell.revealed || cell.flagged) continue;
  cell.revealed = true;
  if (cell.adjacent === 0 && !cell.mine) {
    for (let dr = -1; dr <= 1; dr++) {
      for (let dc = -1; dc <= 1; dc++) {
        const nr = rr + dr;
        const nc = cc + dc;
        if (nr >= 0 && nr < rowsL && nc >= 0 && nc < colsL) stack.push([nr, nc]);
      }
    }
  }
}

}

function handleReveal(r, c) { if (lost || won) return; let b = board; if (!started) { // regenerate board to guarantee first click safe b = generateBoard(rows, cols, mines, [r, c]); setStarted(true); } else { b = cloneBoard(board); }

const cell = b[r][c];
if (cell.revealed || cell.flagged) return;

if (cell.mine) {
  // reveal all mines
  for (let rr = 0; rr < rows; rr++) for (let cc = 0; cc < cols; cc++) if (b[rr][cc].mine) b[rr][cc].revealed = true;
  setBoard(b);
  setLost(true);
  setAiRunning(false);
  return;
}

reveal(b, r, c);
setBoard(b);

// check win: all non-mine cells revealed
let ok = true;
for (let rr = 0; rr < rows; rr++) {
  for (let cc = 0; cc < cols; cc++) {
    if (!b[rr][cc].mine && !b[rr][cc].revealed) ok = false;
  }
}
if (ok) {
  setWon(true);
  setAiRunning(false);
}

}

function handleFlag(r, c, e) { e.preventDefault(); if (lost || won) return; const b = cloneBoard(board); const cell = b[r][c]; if (cell.revealed) return; if (cell.flagged) { cell.flagged = false; setFlagsLeft((f) => f + 1); } else { if (flagsLeft <= 0) return; cell.flagged = true; setFlagsLeft((f) => f - 1); } setBoard(b); }

// AI logic: deterministic rules (simple solver) function aiStep() { // returns true if made progress const b = cloneBoard(board); const R = rows, C = cols; let progress = false;

// rule 1: if revealed number equals flagged neighbors -> reveal other neighbors
for (let r = 0; r < R; r++) {
  for (let c = 0; c < C; c++) {
    const cell = b[r][c];
    if (!cell.revealed || cell.adjacent === 0) continue;
    let flagged = 0, hidden = 0;
    const hiddenCells = [];
    for (let dr = -1; dr <= 1; dr++) for (let dc = -1; dc <= 1; dc++) {
      const nr = r + dr, nc = c + dc;
      if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
      if (nr === r && nc === c) continue;
      const n = b[nr][nc];
      if (n.flagged) flagged++;
      if (!n.revealed && !n.flagged) { hidden++; hiddenCells.push([nr, nc]); }
    }
    if (hidden === 0) continue;

    // if flagged equals number -> reveal hidden
    if (flagged === cell.adjacent) {
      hiddenCells.forEach(([nr, nc]) => {
        if (!b[nr][nc].revealed && !b[nr][nc].flagged) {
          // reveal
          reveal(b, nr, nc);
          progress = true;
        }
      });
    }

    // if flagged + hidden === number -> flag all hidden
    if (flagged + hidden === cell.adjacent) {
      hiddenCells.forEach(([nr, nc]) => {
        if (!b[nr][nc].flagged) {
          b[nr][nc].flagged = true;
          progress = true;
        }
      });
    }
  }
}

if (progress) {
  setBoard(b);
  // update flags count
  const flags = b.flat().filter((x) => x.flagged).length;
  setFlagsLeft(mines - flags);
  return true;
}

// no deterministic progress -> pick a random unopened safe-seeming cell
const unknowns = [];
for (let r = 0; r < R; r++) for (let c = 0; c < C; c++) {
  const cell = b[r][c];
  if (!cell.revealed && !cell.flagged) unknowns.push([r, c]);
}
if (unknowns.length === 0) return false;
const choice = unknowns[Math.floor(Math.random() * unknowns.length)];
const [cr, cc] = choice;

// simulate click
if (!started) {
  // ensure safe first click
  const fresh = generateBoard(rows, cols, mines, [cr, cc]);
  setBoard(fresh);
  setStarted(true);
  reveal(fresh, cr, cc);
  setBoard(cloneBoard(fresh));
  return true;
}

// if the chosen cell is a mine, we will lose — but that's how it is for random fallback
const isMine = board[cr][cc].mine;
if (isMine) {
  // reveal all
  const b2 = cloneBoard(board);
  for (let r = 0; r < R; r++) for (let c = 0; c < C; c++) if (b2[r][c].mine) b2[r][c].revealed = true;
  setBoard(b2);
  setLost(true);
  setAiRunning(false);
  return true;
} else {
  const b2 = cloneBoard(board);
  reveal(b2, cr, cc);
  setBoard(b2);
  // check win
  let ok = true;
  for (let rr = 0; rr < R; rr++) for (let cc2 = 0; cc2 < C; cc2++) if (!b2[rr][cc2].mine && !b2[rr][cc2].revealed) ok = false;
  if (ok) { setWon(true); setAiRunning(false); }
  return true;
}

}

// start/stop AI useEffect(() => { if (aiRunning) { // run ai step every 250ms aiIntervalRef.current = setInterval(() => { aiStep(); }, 250); } else { if (aiIntervalRef.current) { clearInterval(aiIntervalRef.current); aiIntervalRef.current = null; } } return () => { if (aiIntervalRef.current) { clearInterval(aiIntervalRef.current); aiIntervalRef.current = null; } }; // eslint-disable-next-line react-hooks/exhaustive-deps }, [aiRunning, board, started, lost, won]);

// small helpers for render const cellStyle = (cell) => { if (cell.revealed) return "bg-gray-100 border-gray-200"; return "bg-gray-300 hover:bg-gray-200 border-gray-400 cursor-pointer"; };

// UI return ( <div className="max-w-4xl mx-auto p-4"> <div className="flex items-center justify-between mb-4"> <div> <h1 className="text-2xl font-semibold">Mines — Playable Online with AI Bot</h1> <p className="text-sm text-gray-600">Click to reveal, right-click to flag. Toggle the AI to watch it play.</p> </div> <div className="flex gap-2 items-center"> <div className="text-sm">Time: <span className="font-mono">{elapsed}s</span></div> <div className="text-sm">Flags: <span className="font-mono">{flagsLeft}</span></div> <button onClick={() => newGame(rows, cols, mines)} className="px-3 py-1 rounded-lg shadow-sm bg-white border" > New Game </button> </div> </div>

<div className="flex flex-col md:flex-row gap-4">
    <div className="bg-white p-3 rounded-lg shadow-sm">
      {/* Controls */}
      <div className="flex items-center gap-2 mb-3">
        <label className="text-sm">Difficulty</label>
        <select
          value={`${rows}x${cols}:${mines}`}
          onChange={(e) => {
            const val = e.target.value;
            if (val === "Custom") return;
            const [rc, m] = val.split(":");
            const [r, c] = rc.split("x").map(Number);
            setRows(r); setCols(c); setMines(Number(m));
            newGame(r, c, Number(m));
          }}
          className="border rounded px-2 py-1 text-sm"
        >
          {Object.entries(presets).map(([k, v]) => (
            <option key={k} value={`${v.rows}x${v.cols}:${v.mines}`}>{k} ({v.rows}×{v.cols}, {v.mines} mines)</option>
          ))}
          <option value="Custom">Custom</option>
        </select>

        <label className="text-sm">Rows</label>
        <input type="number" min={5} max={30} value={rows} onChange={(e) => setRows(Number(e.target.value))} className="w-16 border rounded px-1 py-1 text-sm" />
        <label className="text-sm">Cols</label>
        <input type="number" min={5} max={60} value={cols} onChange={(e) => setCols(Number(e.target.value))} className="w-16 border rounded px-1 py-1 text-sm" />
        <label className="text-sm">Mines</label>
        <input type="number" min={1} max={Math.max(1, Math.floor(rows * cols * 0.6))} value={mines} onChange={(e) => setMines(Number(e.target.value))} className="w-20 border rounded px-1 py-1 text-sm" />
        <button
          onClick={() => newGame(rows, cols, mines)}
          className="ml-2 px-3 py-1 rounded bg-blue-600 text-white text-sm"
        >Restart</button>
      </div>

      <div className="flex gap-2 items-center">
        <button
          onClick={() => setAiRunning((s) => !s)}
          className={`px-3 py-1 rounded ${aiRunning ? 'bg-red-500 text-white' : 'bg-green-500 text-white'}`}
        >
          {aiRunning ? 'Stop AI' : 'Start AI'}
        </button>
        <button onClick={() => { setAiRunning(false); aiStep(); }} className="px-3 py-1 rounded bg-gray-200">AI Step</button>
        <div className="text-sm text-gray-600 ml-2">Status: {lost ? 'Lost' : won ? 'Won' : started ? 'Playing' : 'Ready'}</div>
      </div>

    </div>

    <div className="flex-1 bg-white p-3 rounded-lg shadow-sm overflow-auto">
      <div style={{ width: cols * 34 }} className="inline-block">
        <div className="grid" style={{ gridTemplateColumns: `repeat(${cols}, 34px)`, gap: 2 }}>
          {board.flat().map((cell, idx) => {
            const r = Math.floor(idx / cols);
            const c = idx % cols;
            return (
              <div
                key={`${r}-${c}`}
                onClick={() => handleReveal(r, c)}
                onContextMenu={(e) => handleFlag(r, c, e)}
                className={`w-8 h-8 flex items-center justify-center text-sm select-none rounded ${cell.revealed ? 'cursor-default' : ''} ${cellStyle(cell)}`}
                title={cell.revealed && cell.adjacent > 0 ? `Adjacent: ${cell.adjacent}` : ''}
              >
                {cell.revealed ? (
                  cell.mine ? (
                    <span className="text-red-600">💣</span>
                  ) : (
                    cell.adjacent > 0 ? <span className="font-semibold">{cell.adjacent}</span> : <span className="text-gray-400">&nbsp;</span>
                  )
                ) : (cell.flagged ? <span>🚩</span> : null)}
              </div>
            );
          })}
        </div>
      </div>

      <div className="mt-3 text-xs text-gray-600">
        <strong>Controls:</strong> Left click reveal | Right click to flag. AI uses simple Minesweeper inference (if number = flagged neighbors, it opens the rest; if number = flagged+hidden, it flags hidden). When stuck it picks a random unknown cell.
      </div>
    </div>
  </div>

  <div className="mt-4 text-sm text-gray-700">
    Tip: For the best first-move experience, let the AI make the first click (it avoids immediate losses by regenerating with a safe first click).
  </div>
</div>

); }

 