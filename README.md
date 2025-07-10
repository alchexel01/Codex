import pygame
import numpy as np
import sys

# Initialize Pygame
pygame.init()
pygame.font.init()

# Screen dimensions
WIDTH, HEIGHT = 800, 600
screen = pygame.display.set_mode((WIDTH, HEIGHT))
pygame.display.set_caption("Matrix Solver - Modern Pygame App")

# Colors (customizable)
COLOR_BG = (18, 18, 30)
COLOR_TEXT = (220, 220, 230)
COLOR_ACCENT = (72, 209, 204)
COLOR_ACCENT2 = (138, 43, 226)
COLOR_INPUT_BG = (30, 30, 45)
COLOR_BUTTON_BG = (40, 40, 60)
COLOR_BUTTON_HOVER = (72, 209, 204)
COLOR_ERROR = (255, 70, 70)

# Fonts
FONT_MAIN = pygame.font.SysFont("Segoe UI", 24)
FONT_SMALL = pygame.font.SysFont("Segoe UI", 25)
FONT_R = pygame.font.SysFont("Segoe UI", 30)
FONT_M = pygame.font.SysFont("Segoe UI", 10)
FONT_LARGE = pygame.font.SysFont("Segoe UI", 25, bold=True)

# Matrix grid size defaults
ROWS, COLS = 3, 3
CELL_SIZE = 75
GRID_ORIGIN = (50, 100)

# Virtual keyboard layout
KEYS = [
    ['7', '8', '9'],
    ['4', '5', '6'],
    ['1', '2', '3'],
    ['0', '-', 'Del']
]

MAX_ROWS, MAX_COLS = 5, 5  # Maximum allowed rows and columns

class Button:
    def __init__(self, rect, text, color_bg=COLOR_BUTTON_BG, color_hover=COLOR_BUTTON_HOVER):
        self.rect = pygame.Rect(rect)
        self.text = text
        self.color_bg = color_bg
        self.color_hover = color_hover
        self.hovered = False

    def draw(self, surface):
        color = self.color_hover if self.hovered else self.color_bg
        pygame.draw.rect(surface, color, self.rect, border_radius=6)
        txt_surf = FONT_MAIN.render(self.text, True, COLOR_TEXT)
        txt_rect = txt_surf.get_rect(center=self.rect.center)
        surface.blit(txt_surf, txt_rect)

    def check_hover(self, pos):
        self.hovered = self.rect.collidepoint(pos)

    def clicked(self, pos):
        return self.rect.collidepoint(pos)

class Selector:
    def __init__(self, label, options, position, selected=3):
        self.label = label
        self.options = options
        self.position = position
        self.selected = selected
        self.buttons = []
        self.create_buttons()

    def create_buttons(self):
        x, y = self.position
        btn_w, btn_h = 40, 30
        gap = 5
        self.buttons.clear()
        for i, val in enumerate(self.options):
            rect = pygame.Rect(x + i * (btn_w + gap), y, btn_w, btn_h)
            self.buttons.append(Button(rect, str(val), COLOR_BUTTON_BG, COLOR_BUTTON_HOVER))

    def draw(self, surface):
        label_surf = FONT_M.render(self.label, True, COLOR_ACCENT)
        surface.blit(label_surf, (self.position[0], self.position[1] - 12))
        for btn in self.buttons:
            btn.hovered = (btn.text == str(self.selected))
            btn.draw(surface)

    def handle_click(self, pos):
        for btn in self.buttons:
            if btn.clicked(pos):
                self.selected = int(btn.text)
                return True
        return False

class MatrixInput:
    def __init__(self, rows, cols, origin, cell_size):
        self.rows = rows
        self.cols = cols
        self.origin = origin
        self.cell_size = cell_size
        self.cells = [['' for _ in range(cols)] for _ in range(rows)]
        self.selected = (0, 0)

    def draw(self, surface):
        ox, oy = self.origin
        for r in range(self.rows):
            for c in range(self.cols):
                rect = pygame.Rect(ox + c * self.cell_size, oy + r * self.cell_size,
                                   self.cell_size, self.cell_size)
                if (r, c) == self.selected:
                    pygame.draw.rect(surface, COLOR_ACCENT2, rect, border_radius=5)
                else:
                    pygame.draw.rect(surface, COLOR_INPUT_BG, rect, border_radius=5)
                pygame.draw.rect(surface, COLOR_ACCENT, rect, 2, border_radius=5)
                val = self.cells[r][c]
                txt_surf = FONT_MAIN.render(val, True, COLOR_TEXT)
                txt_rect = txt_surf.get_rect(center=rect.center)
                surface.blit(txt_surf, txt_rect)

    def select_cell(self, pos):
        ox, oy = self.origin
        x, y = pos
        if ox <= x < ox + self.cols * self.cell_size and oy <= y < oy + self.rows * self.cell_size:
            c = (x - ox) // self.cell_size
            r = (y - oy) // self.cell_size
            self.selected = (r, c)
            return True
        return False

    def input_value(self, char):
        r, c = self.selected
        current = self.cells[r][c]
        if char == 'Del':
            self.cells[r][c] = current[:-1]
        else:
            if char == '.' and '.' in current:
                return
            if len(current) < 6:
                self.cells[r][c] = current + char

    def get_matrix(self):
        try:
            mat = np.array([[float(cell) if cell != '' else 0 for cell in row] for row in self.cells])
            return mat
        except ValueError:
            return None

class MatrixSolverApp:
    def __init__(self):
        self.running = True
        self.clock = pygame.time.Clock()

        self.m1_rows, self.m1_cols = 3, 3
        self.m2_rows, self.m2_cols = 3, 3

        self.m1_row_selector = Selector("Matrix 1 Rows", list(range(1, MAX_ROWS + 1)), (50, 60), self.m1_rows)
        self.m1_col_selector = Selector("Matrix 1 Columns", list(range(1, MAX_COLS + 1)), (50, 20), self.m1_cols)
        self.m2_row_selector = Selector("Matrix 2 Rows", list(range(1, MAX_ROWS + 1)), (450, 60), self.m2_rows)
        self.m2_col_selector = Selector("Matrix 2 Columns", list(range(1, MAX_COLS + 1)), (450, 20), self.m2_cols)

        self.matrix1 = MatrixInput(self.m1_rows, self.m1_cols, (50, 300), CELL_SIZE)
        self.matrix2 = MatrixInput(self.m2_rows, self.m2_cols, (450, 300), CELL_SIZE)

        self.keyboard_buttons = []
        key_w = 200
        key_h = 100
        start_x = 60
        start_y = 1100
        for row_idx, row in enumerate(KEYS):
            for col_idx, key in enumerate(row):
                rect = (start_x + col_idx * key_w, start_y + row_idx * key_h, key_w - 10, key_h - 10)
                self.keyboard_buttons.append(Button(rect, key))

        op_names = ["Add", "Subtract", "Multiply", "Trans. M1", "Trans. M2", "Dtrmt. M1", "Dtrmt. M2", "Inverse M1", "Inverse M2"]
        self.op_buttons = []
        op_x = 520
        op_y = 600
        op_w = 140
        op_h = 40
        gap = 10
        for i, op in enumerate(op_names):
            rect = (op_x, op_y + i * (op_h + gap), op_w, op_h)
            self.op_buttons.append(Button(rect, op, COLOR_ACCENT2, COLOR_ACCENT))

        self.result_lines = []
        self.error_message = ""
        self.active_matrix = 1

    def draw_labels(self):
        title1 = FONT_LARGE.render("", True, COLOR_ACCENT)
        title2 = FONT_LARGE.render("", True, COLOR_ACCENT)
        screen.blit(title1, (50, 250))
        screen.blit(title2, (450, 250))
        active_text = FONT_SMALL.render(f"Active Input: Matrix {self.active_matrix}", True, COLOR_ACCENT2)
        screen.blit(active_text, (50, 600))
        result_label = FONT_LARGE.render("Result:", True, COLOR_ACCENT)
        screen.blit(result_label, (50, 700))

    def draw_result(self):
        y = 760
        for line in self.result_lines:
            txt_surf = FONT_R.render(line, True, COLOR_TEXT)
            screen.blit(txt_surf, (50, y))
            y += 50

    def run(self):
        while self.running:
            self.clock.tick(60)
            screen.fill(COLOR_BG)

            self.m1_row_selector.draw(screen)
            self.m1_col_selector.draw(screen)
            self.m2_row_selector.draw(screen)
            self.m2_col_selector.draw(screen)

            self.matrix1.draw(screen)
            self.matrix2.draw(screen)
            self.draw_labels()

            for btn in self.keyboard_buttons:
                btn.draw(screen)
            for btn in self.op_buttons:
                btn.draw(screen)

            self.draw_result()

            pos = pygame.mouse.get_pos()
            for btn in self.keyboard_buttons + self.op_buttons:
                btn.check_hover(pos)

            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    self.running = False

                elif event.type == pygame.MOUSEBUTTONDOWN:
                    if self.m1_row_selector.handle_click(pos):
                        self.m1_rows = self.m1_row_selector.selected
                        self.matrix1 = MatrixInput(self.m1_rows, self.m1_cols, (50, 100), CELL_SIZE)
                        self.active_matrix = 1
                        continue
                    if self.m1_col_selector.handle_click(pos):
                        self.m1_cols = self.m1_col_selector.selected
                        self.matrix1 = MatrixInput(self.m1_rows, self.m1_cols, (50, 100), CELL_SIZE)
                        self.active_matrix = 1
                        continue
                    if self.m2_row_selector.handle_click(pos):
                        self.m2_rows = self.m2_row_selector.selected
                        self.matrix2 = MatrixInput(self.m2_rows, self.m2_cols, (450, 100), CELL_SIZE)
                        self.active_matrix = 2
                        continue
                    if self.m2_col_selector.handle_click(pos):
                        self.m2_cols = self.m2_col_selector.selected
                        self.matrix2 = MatrixInput(self.m2_rows, self.m2_cols, (450, 100), CELL_SIZE)
                        self.active_matrix = 2
                        continue

                    if self.matrix1.select_cell(pos):
                        self.active_matrix = 1
                    elif self.matrix2.select_cell(pos):
                        self.active_matrix = 2
                    else:
                        for btn in self.keyboard_buttons:
                            if btn.clicked(pos):
                                if self.active_matrix == 1:
                                    self.matrix1.input_value(btn.text)
                                else:
                                    self.matrix2.input_value(btn.text)
                                break
                        for btn in self.op_buttons:
                            if btn.clicked(pos):
                                self.perform_operation(btn.text)
                                break

            pygame.display.flip()

        pygame.quit()
        sys.exit()

    def perform_operation(self, op_name):
        op = op_name.strip().lower()
        m1 = self.matrix1.get_matrix()
        m2 = self.matrix2.get_matrix()
        self.result_lines = []
        self.error_message = ""

        try:
            if m1 is None or m2 is None:
                self.error_message = "Invalid number format in matrices."
                self.result_lines = [self.error_message]
                return

            if op == "add":
                if m1.shape != m2.shape:
                    self.error_message = "Matrix shapes must match for addition."
                else:
                    res = np.add(m1, m2)
                    self.result_lines = self.format_matrix(res)

            elif op == "subtract":
                if m1.shape != m2.shape:
                    self.error_message = "Matrix shapes must match for subtraction."
                else:
                    res = np.subtract(m1, m2)  # Corrected subtraction
                    self.result_lines = self.format_matrix(res)

            elif op == "multiply":
                if m1.shape[1] != m2.shape[0]:
                    self.error_message = "Matrix shapes incompatible for multiplication."
                else:
                    res = np.matmul(m1, m2)
                    self.result_lines = self.format_matrix(res)

            elif op == "trans. m1":
                res = np.transpose(m1)
                self.result_lines = self.format_matrix(res)

            elif op == "trans. m2":
                res = np.transpose(m2)
                self.result_lines = self.format_matrix(res)

            elif op == "dtrmt. m1":
                if m1.shape[0] != m1.shape[1]:
                    self.error_message = "Matrix 1 must be square for determinant."
                else:
                    det = np.linalg.det(m1)
                    self.result_lines = [f"Determinant: {det:.4f}"]

            elif op == "dtrmt. m2":
                if m2.shape[0] != m2.shape[1]:
                    self.error_message = "Matrix 2 must be square for determinant."
                else:
                    det = np.linalg.det(m2)
                    self.result_lines = [f"Determinant: {det:.4f}"]

            elif op == "inverse m1":
                if m1.shape[0] != m1.shape[1]:
                    self.error_message = "Matrix 1 must be square for inverse."
                else:
                    inv = np.linalg.inv(m1)
                    self.result_lines = self.format_matrix(inv)

            elif op == "inverse m2":
                if m2.shape[0] != m2.shape[1]:
                    self.error_message = "Matrix 2 must be square for inverse."
                else:
                    inv = np.linalg.inv(m2)
                    self.result_lines = self.format_matrix(inv)

            else:
                self.error_message = "Unknown Operation."

            if self.error_message:
                self.result_lines = [self.error_message]

        except np.linalg.LinAlgError:
            self.result_lines = ["Singular/not invertible."]
        except Exception as e:
            self.result_lines = [f"Error: {str(e)}"]

    @staticmethod
    def format_matrix(mat):
        lines = []
        for row in mat:
            line = "  ".join(f"{v:.3g}" for v in row)
            lines.append(line)
        return lines

if __name__ == "__main__":
    app = MatrixSolverApp()
    app.run()
    
