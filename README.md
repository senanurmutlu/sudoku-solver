import tkinter as tk
from tkinter import filedialog, messagebox
import matplotlib.pyplot as plt
from pysat.solvers import Glucose3
import google.generativeai as genai
import json
import re
from PIL import Image
import os 

genai.configure(api_key=GEMINI_API_KEY)
#kullanılacak model  
model = genai.GenerativeModel("gemini-2.5-flash")

def image_to_grid(image_path):
    """
    Verilen Sudoku görselini Gemini kullanarak 9x9 grid'e çevirir.
    Boş hücreler 0 olarak döner.
    """
    image = Image.open(image_path)

    # Gemini'ye verilecek prompt
    prompt = (
        "Extract the Sudoku grid.\n"
        "Return a 9x9 matrix using numbers 0-9.\n"
        "0 means empty cell."
    )

    # Görsel + prompt gönder
    response = model.generate_content([prompt, image])
    raw = response.text

    # JSON formatını regex ile yakala
    match = re.search(r"\[\s*\[.*?\]\s*\]", raw, re.DOTALL)

    if not match:
        raise ValueError(
            "OCR JSON üretmedi.\n\nGemini çıktısı:\n" + raw
        )

    json_text = match.group(0)

    # JSON parse et
    try:
        grid = json.loads(json_text)
    except json.JSONDecodeError:
        raise ValueError("JSON parse edilemedi:\n" + json_text)

    # 9x9 kontrolü
    if len(grid) != 9 or any(len(row) != 9 for row in grid):
        raise ValueError("OCR sonucu 9x9 değil.")

    return grid

def var(i, j, n):
    """
    (i, j) hücresinde n sayısının olması durumunu
    tek bir değişken numarasına map eder (SAT encoding için).
    """
    return 81 * i + 9 * j + n


def encode_sudoku(grid):
    """
    Sudoku grid'ini CNF formatına çevirir.
    (SAT solver için gerekli)
    """
    cnf = []

    # Her hücrede en az bir sayı olmalı
    for i in range(9):
        for j in range(9):
            cnf.append([var(i, j, n) for n in range(1, 10)])

            # Aynı hücrede iki sayı olamaz
            for n1 in range(1, 10):
                for n2 in range(n1 + 1, 10):
                    cnf.append([-var(i, j, n1), -var(i, j, n2)])

    # Her satırda her sayı bir kez bulunmalı
    for i in range(9):
        for n in range(1, 10):
            cnf.append([var(i, j, n) for j in range(9)])

    # Her sütunda her sayı bir kez bulunmalı
    for j in range(9):
        for n in range(1, 10):
            cnf.append([var(i, j, n) for i in range(9)])

    # Her 3x3 blokta her sayı bir kez bulunmalı
    for bi in range(3):
        for bj in range(3):
            for n in range(1, 10):
                block = []
                for i in range(bi * 3, bi * 3 + 3):
                    for j in range(bj * 3, bj * 3 + 3):
                        block.append(var(i, j, n))
                cnf.append(block)

    # Başlangıç gridindeki verilen sayılar
    for i in range(9):
        for j in range(9):
            if grid[i][j] != 0:
                cnf.append([var(i, j, grid[i][j])])

    return cnf


def solve_all_solutions(cnf):
    """
    Verilen CNF için tüm çözümleri bulur.
    SAT solver kullanır (Glucose3).
    """
    solver = Glucose3()

    # CNF clause'larını solver'a ekle
    for clause in cnf:
        solver.add_clause(clause)

    solutions = []

    # Tüm çözümleri bul
    while solver.solve():
        model = solver.get_model()
        solution = [[0] * 9 for _ in range(9)]

        # Modeli Sudoku gridine çevir
        for v in model:
            if v > 0:
                v -= 1
                i = v // 81
                j = (v % 81) // 9
                n = (v % 9) + 1
                solution[i][j] = n

        solutions.append(solution)

        # Aynı çözümü tekrar bulmamak için engelle
        solver.add_clause([-v for v in model if v > 0])

    return solutions


def draw_sudoku(grid, title):
    """
    Sudoku çözümünü matplotlib ile görselleştirir.
    """
    fig, ax = plt.subplots(figsize=(4, 4))
    ax.set_title(title)

    ax.set_xticks(range(10))
    ax.set_yticks(range(10))
    ax.grid(True)

    # Sayıları hücrelere yaz
    for i in range(9):
        for j in range(9):
            ax.text(j + 0.5, 8.5 - i, str(grid[i][j]),
                    ha='center', va='center', fontsize=14)

    ax.set_xticklabels([])
    ax.set_yticklabels([])

    plt.show()


def open_image():
    """
    Kullanıcıdan Sudoku görseli alır,
    çözer ve sonuçları gösterir.
    """
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
            messagebox.showinfo("Sonuç", "Sudoku çözülemez.")
            return

      messagebox.showinfo("Sonuç", f"{len(solutions)} çözüm bulundu.")

      for i, sol in enumerate(solutions):
            draw_sudoku(sol, f"Çözüm {i + 1}")

   except Exception as e:
        messagebox.showerror("Hata", str(e))

def main():
    root = tk.Tk()
    root.title("Sudoku SAT Solver (Gemini API)")

  tk.Label(
        root,
        text="SUDOKU SAT SOLVER",
        font=("Arial", 16),
        fg='purple'
    ).pack(pady=10)

   tk.Button(
        root,
        text="Sudoku Görseli Yükle",
        command=open_image,
        width=30,
        height=2,
        fg='purple'
   ).pack(pady=20)

   root.mainloop()  
    if __name__ == "__main__":
    main()
