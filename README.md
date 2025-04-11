import tkinter as tk
from tkinter import ttk
import tkinter.font as tkFont

# 샘플 화학 반응 데이터
data = [
    ("1", "2"),
    ("3", "4"),
    ("5", "6"),
    ("7", "8l"),
    ("9", "10")
]

# Tkinter 창 생성
root = tk.Tk()
root.title("화학 반응 예측 프로그램")
root.geometry("600x400")  # 창 크기 설정

# 기본 폰트 설정
default_font = tkFont.Font(family="맑은 고딕", size=14)

# Treeview 위젯 생성 (반응물 목록 표시)
tree = ttk.Treeview(root, columns=("Reactant",), show="headings", height=5)
tree.heading("Reactant", text="반응물")

# 열 너비 설정
tree.column("Reactant", width=200, anchor="center")

# 데이터 삽입 (반응물만 표시)
for reactant, _ in data:
    tree.insert("", "end", values=(reactant,))

tree.pack(expand=True, fill="both", pady=10)

# 선택한 반응물에 대한 생성물 표시 함수
def on_item_double_clicked(event):
    selected_item = tree.selection()
    if selected_item:
        reactant = tree.item(selected_item[0], "values")[0]
        for r, p in data:
            if r == reactant:
                # 새로운 창 생성
                product_window = tk.Toplevel(root)
                product_window.title("생성물 정보")
                product_window.geometry("300x100")
                tk.Label(product_window, text=f"반응물: {reactant}", font=default_font).pack(pady=10)
                tk.Label(product_window, text=f"생성물: {p}", font=default_font).pack(pady=10)
                break

# Treeview에 더블클릭 이벤트 바인딩
tree.bind("<Double-1>", on_item_double_clicked)

# 사용자 입력을 위한 프레임 생성
input_frame = tk.Frame(root)
input_frame.pack(pady=10)

# 반응물 입력 라벨
input_label = tk.Label(input_frame, text="새로운 반응물 입력:", font=default_font)
input_label.pack(side="left", padx=5)

# 반응물 입력 엔트리
reactant_entry = tk.Entry(input_frame, width=30, font=default_font)
reactant_entry.pack(side="left", padx=5)

# 예측 결과 표시 라벨
predict_result_label = tk.Label(root, text="예상 생성물: ", font=default_font)
predict_result_label.pack(pady=10)

# 예측 함수 정의
def predict_product():
    reactant = reactant_entry.get()
    for r, p in data:
        if r == reactant:
            predict_result_label.config(text=f"예상 생성물: {p}")
            return
    predict_result_label.config(text="예상 생성물: 모르는 반응입니다.")

# 예측 버튼 생성
predict_button = tk.Button(input_frame, text="예측", command=predict_product, font=default_font)
predict_button.pack(side="left", padx=5)

root.mainloop()
