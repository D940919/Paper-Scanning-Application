# Paper-Scanning-Application
This is an application that scan image of paper
Code

import cv2
import numpy as np
from tkinter import Tk
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
        if w > 20 and h > 20:  
            detected_answers.append((x, y))
    
    return detected_answers

def map_answers_to_options(detected_answers, options_map):
    
    mapped_answers = []
    for x, y in detected_answers:
        for key, value in options_map.items():
            if value[0] <= x <= value[2] and value[1] <= y <= value[3]:
                mapped_answers.append(key)
    return mapped_answers

def compare_and_annotate(image, mapped_answers, correct_answers, options_map):

    annotated_image = image.copy()
    
    for i, answer in enumerate(mapped_answers):
        if i < len(correct_answers) and correct_answers[i] == answer:
            cv2.putText(annotated_image, 'Correct', options_map[answer][:2], cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 255, 0), 2)
        else:
            cv2.putText(annotated_image, 'Wrong', options_map[answer][:2], cv2.FONT_HERSHEY_SIMPLEX, 0.5, (0, 0, 255), 2)
    
    return annotated_image

def generate_output(annotated_image, correct_answers):
    cv2.imshow('Checked Paper', annotated_image)
    cv2.imwrite('checked_paper.png', annotated_image)
    
    print("Correct Answers List:")
    for idx, answer in enumerate(correct_answers):
        print(f"Q{idx+1}: {answer}")

def process_mcq_image(correct_answers, options_map):
    image_path = select_image_from_pc()

    original_image = cv2.imread(image_path)
    
    if original_image is None:
        raise FileNotFoundError(f"Image file not found at path: {image_path}")

    binary_image = preprocess_image(original_image)

    detected_answers = detect_answers(binary_image)
    mapped_answers = map_answers_to_options(detected_answers, options_map)
    annotated_image = compare_and_annotate(original_image, mapped_answers, correct_answers, options_map)
    
    generate_output(annotated_image, correct_answers)
    
options_map = {
    'A': (100, 150, 120, 170),
    'B': (200, 150, 220, 170),
    'C': (300, 150, 320, 170),
    'D': (400, 150, 420, 170)
}

correct_answers = ['A', 'C', 'B', 'D', 'A']
process_mcq_image(correct_answers, options_map)

