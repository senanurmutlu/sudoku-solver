import tkinter as tk
from tkinter import filedialog, messagebox
import matplotlib.pyplot as plt
from pysat.solvers import Glucose3
import google.generativeai as genai
import json
import re
from PIL import Image
import os

# -------------------------
# API SETUP
# -------------------------
GEMINI_API_KEY = os.getenv("GEMINI_API_KEY")
genai.configure(api_key=GEMINI_API_KEY)

model = genai.GenerativeModel("gemini-2.5-flash")


# -------------------------
# OCR: IMAGE -> GRID
# -------------------------
def image_to_grid(image_path):
    image = Image.open(image_path)

    prompt = (
        "Extract the Sudoku grid.\n"
        "Return ONLY a valid 9x9 JSON array.\n"
        "Use numbers 0-9, where 0 = empty cell."
    )

    response = model.generate_content([prompt, image])
    raw = response.text

    match = re.search(r"\[\s*\[.*?\]\s*\]", raw, re.DOTALL)

    if not match:
        raise ValueError("OCR JSON üretmedi:\n" + raw)

    json_text = match.group(0)

    grid = json.loads(json_text)

    if len(grid) != 9 or any(len(row) != 9 for row in grid):
        raise ValueError("Grid 9x9 değil.")

    return grid


# -------------------------
# SAT VARIABLE MAPPING
# -------------------------
def var(i, j, n):
    return 81 * i + 9 * j + n


# -------------------------
# CNF ENCODING
# -------------------------
def encode_sudoku(grid):
    cnf = []

    # Her hücre en az 1 sayı
    for i in range(9):
        for j in range(9):
            cnf.append([var(i, j, n) for n in range(1, 10)])

            # En fazla 1 sayı
            for n1 in range(1, 10):
                for n2 in range(n1 + 1, 10):
                    cnf.append([-var(i, j, n1), -var(i, j, n2)])

    # Satırlar
    for i in range(9):
        for n in range(1, 10):
            cnf.append([var(i, j, n) for j in range(9)])

    # Sütunlar
    for j in range(9):
        for n in range(1, 10):
            cnf.append([var(i, j, n) for i in range(9)])

    # 3x3 bloklar
    for bi in range(3):
        for bj in range(3):
            for n in range(1, 10):
                block = []
                for i in range(bi * 3, bi * 3 + 3):
                    for j in range(bj * 3, bj * 3 + 3):
                        block.append(var(i, j, n))
                cnf.append(block)

    # Başlangıç değerleri
    for i in range(9):
        for j in range(9):
            if grid[i][j] != 0:
                cnf.append([var(i, j, grid[i][j])])

    return cnf


# -------------------------
# SOLVER
# -------------------------
def solve_all_solutions(cnf):
    solver = Glucose3()

    for clause in cnf:
        solver.add_clause(clause)

    solutions = []

    while solver.solve():
        model = solver.get_model()
        solution = [[0] * 9 for _ in range(9)]

        for v in model:
            if v > 0:
                v -= 1
                i = v // 81
                j = (v % 81) // 9
                n = (v % 9) + 1
                solution[i][j] = n

        solutions.append(solution)

        solver.add_clause([-v for v in model if v > 0])

    return solutions


# -------------------------
# DRAW SUDOKU
# -------------------------
def draw_sudoku(grid, title):
    fig, ax = plt.subplots(figsize=(4, 4))
    ax.set_title(title)

    ax.set_xticks(range(10))
    ax.set_yticks(range(10))
    ax.grid(True)

    for i in range(9):
        for j in range(9):
            ax.text(j + 0.5, 8.5 - i, str(grid[i][j]),
                    ha='center', va='center', fontsize=14)

    ax.set_xticklabels([])
    ax.set_yticklabels([])

    plt.show()


# -------------------------
# GUI FUNCTION
# -------------------------
def open_image():
    try:
        path = filedialog.askopenfilename(
            title="Sudoku Görseli Seç",
            filetypes=[("Image Files", "*.png *.jpg *.jpeg")]
        )

        if not path:
            return

        grid = image_to_grid(path)
        cnf = encode_sudoku(grid)
        solutions = solve_all_solutions(cnf)

        if not solutions:
            messagebox.showinfo("Sonuç", "Çözüm bulunamadı.")
            return

        messagebox.showinfo("Sonuç", f"{len(solutions)} çözüm bulundu.")

        for i, sol in enumerate(solutions):
            draw_sudoku(sol, f"Çözüm {i + 1}")

    except Exception as e:
        messagebox.showerror("Hata", str(e))


# -------------------------
# MAIN GUI
# -------------------------
def main():
    root = tk.Tk()
    root.title("Sudoku SAT Solver (Gemini AI)")

    tk.Label(
        root,
        text="SUDOKU SAT SOLVER",
        font=("Arial", 16),
        fg="purple"
    ).pack(pady=10)

    tk.Button(
        root,
        text="Sudoku Görseli Yükle",
        command=open_image,
        width=30,
        height=2,
        fg="purple"
    ).pack(pady=20)

    root.mainloop()


# -------------------------
# START PROGRAM
# -------------------------
if __name__ == "__main__":
    main()
