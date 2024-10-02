import cv2
import numpy as np
from tkinter import Tk, Label
from PIL import Image, ImageTk
from tkinter.filedialog import askopenfilename

def select_image_from_pc():
    root = Tk()
    root.withdraw()
    root.lift()
    root.attributes('-topmost', True)

    image_path = askopenfilename(filetypes=[("Image files", "*.jpg;*.jpeg;*.png;*.bmp")])

    root.destroy()

    if not image_path:
        raise Exception("No image selected")

    return image_path

def preprocess_image(image):
    grayscale_image = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
    _, binary_image = cv2.threshold(grayscale_image, 150, 255, cv2.THRESH_BINARY_INV)
    binary_image = cv2.medianBlur(binary_image, 5)

    return binary_image

def detect_answers(binary_image):
    contours, _ = cv2.findContours(binary_image, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    detected_answers = []

    for cnt in contours:
        x, y, w, h = cv2.boundingRect(cnt)
        if 15 < w < 50 and 15 < h < 50:  # Bubble size condition
            detected_answers.append((x, y, w, h))
    
    return detected_answers

def group_by_questions(detected_answers):
    detected_answers.sort(key=lambda pos: pos[1])  # Sort by y-coordinate
    grouped = []
    current_group = []

    for i, (x, y, w, h) in enumerate(detected_answers):
        if i == 0:
            current_group.append((x, y, w, h))
        else:
            # Group bubbles that are horizontally aligned (same question)
            if abs(y - detected_answers[i - 1][1]) < 30:  # Adjust threshold based on your image
                current_group.append((x, y, w, h))
            else:
                grouped.append(current_group)
                current_group = [(x, y, w, h)]

    if current_group:
        grouped.append(current_group)

    print(f"Grouped answers by questions: {grouped}")
    return grouped

def map_answers_to_options(grouped_answers):
    options = ['A', 'B', 'C', 'D', 'E']
    mapped_answers = []

    for idx, group in enumerate(grouped_answers):
        print(f"Question {idx + 1}: Detected Bubbles: {group}")
        group.sort(key=lambda pos: pos[0])  # Sort by x-coordinate to map left-to-right

        # If only one bubble is detected for the question, map it to the corresponding option
        if len(group) > 0:
            # Choose the first bubble in the sorted group as the answer (modify this based on how you want to handle multiple filled bubbles)
            selected_bubble = group[0]  # For now, we assume the first detected bubble is the answer
            bubble_index = group.index(selected_bubble)  # Get the index of the selected bubble
            mapped_answers.append(options[bubble_index])
        else:
            mapped_answers.append(None)  # No answer detected for the question

        print(f"Question {idx + 1}: Answer Mapped: {mapped_answers[-1]}")

    return mapped_answers

def compare_and_annotate(image, mapped_answers, correct_answers):
    annotated_image = image.copy()

    # Calculate the score
    score = 0
    for i, answer in enumerate(mapped_answers):
        print(f"Comparing Question {i + 1}: Your Answer: {answer}, Correct Answer: {correct_answers[i]}")
        if answer == correct_answers[i]:
            score += 1

    total_questions = len(correct_answers)

    # Draw the circle for the score
    circle_center = (80, 50)
    cv2.circle(annotated_image, circle_center, 30, (0, 0, 0), thickness=2)
    
    score_text = f"{score}/{total_questions}"
    text_size, _ = cv2.getTextSize(score_text, cv2.FONT_HERSHEY_SIMPLEX, 0.7, 3)
    text_x = circle_center[0] - text_size[0] // 2
    text_y = circle_center[1] + text_size[1] // 2
    cv2.putText(annotated_image, score_text, (text_x, text_y), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 3)
    
    # Mark answers with ticks, crosses, and question marks
    for i in range(len(mapped_answers)):
        mark_position = (20, (i + 1) * 30 + 50)
        if mapped_answers[i] == correct_answers[i]:
            cv2.putText(annotated_image, "✓", (mark_position[0], mark_position[1]), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
        elif mapped_answers[i] is None:
            cv2.putText(annotated_image, "?", (mark_position[0], mark_position[1]), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)
        else:
            cv2.putText(annotated_image, "✗", (mark_position[0], mark_position[1]), cv2.FONT_HERSHEY_SIMPLEX, 0.7, (0, 0, 255), 2)

    return annotated_image

def generate_output(annotated_image):
    # Convert image to PIL format
    annotated_image_pil = Image.fromarray(cv2.cvtColor(annotated_image, cv2.COLOR_BGR2RGB))

    # Create Tkinter window
    root = Tk()
    root.title("Checked Paper")

    # Convert PIL image to ImageTk format
    image_tk = ImageTk.PhotoImage(annotated_image_pil)

    # Create a label to display the image
    label = Label(root, image=image_tk)
    label.pack(fill='both', expand='yes')

    # Start Tkinter main loop
    root.mainloop()

def process_mcq_image(correct_answers):
    image_path = select_image_from_pc()

    original_image = cv2.imread(image_path)

    if original_image is None:
        raise FileNotFoundError(f"Image file not found at path: {image_path}")

    binary_image = preprocess_image(original_image)

    detected_answers = detect_answers(binary_image)
    grouped_answers = group_by_questions(detected_answers)
    mapped_answers = map_answers_to_options(grouped_answers)

    annotated_image = compare_and_annotate(original_image, mapped_answers, correct_answers)

    generate_output(annotated_image)

# Correct answers input as A, B, C, D
correct_answers = ['A', 'C', 'D', 'A', 'C', 'D', 'A', 'E', 'B', 'D']  # 10 MCQs

process_mcq_image(correct_answers)
