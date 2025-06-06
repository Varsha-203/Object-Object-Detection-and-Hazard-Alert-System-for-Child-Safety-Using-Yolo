# Object-Object-Detection-and-Hazard-Alert-System-for-Child-Safety-Using-Yoloimport cv2
import numpy as np
import urllib.request
import os
import time
from IPython.display import clear_output, display
import PIL.Image
import io

# Download YOLO files
def download_file(url, filename):
    if not os.path.exists(filename):
        print(f"Downloading {filename}...")
        urllib.request.urlretrieve(url, filename)

download_file("https://pjreddie.com/media/files/yolov3.weights", "yolov3.weights")
download_file("https://raw.githubusercontent.com/pjreddie/darknet/master/cfg/yolov3.cfg", "yolov3.cfg")
download_file("https://raw.githubusercontent.com/pjreddie/darknet/master/data/coco.names", "coco.names")

# Load YOLO model
net = cv2.dnn.readNet("yolov3.weights", "yolov3.cfg")
layer_names = net.getLayerNames()
output_layers = [layer_names[i - 1] for i in net.getUnconnectedOutLayers()]

# Load class labels
with open("coco.names", "r") as f:
    classes = [line.strip() for line in f.readlines()]

# Define hazardous and safe objects
hazard_objects = ["knife", "stove", "electrical_wire", "scissors", "dog pen", "gun", "fire"]
safe_objects = ["ball", "teddy bear", "pacifier", "bottle", "diaper", "toy"]

# Function to calculate distance
def calculate_distance(box1, box2):
    x1, y1, w1, h1 = box1
    x2, y2, w2, h2 = box2
    center1 = (x1 + w1 // 2, y1 + h1 // 2)
    center2 = (x2 + w2 // 2, y2 + h2 // 2)
    distance = np.sqrt((center1[0] - center2[0]) * 2 + (center1[1] - center2[1]) * 2)
    return distance

# Alert function for "Danger"
def display_danger_alert(frame):
    cv2.putText(frame, "DANGER", (200, 250), cv2.FONT_HERSHEY_SIMPLEX, 5, (0, 0, 255), 10, cv2.LINE_AA)
    # Stop the video feed
    return True, frame

# Function to check proximity to knife
def check_proximity_to_knife(child_boxes, hazard_boxes, class_ids, frame, alert_triggered):
    for child_box in child_boxes:
        for hazard_box in hazard_boxes:
            # Check if hazard object is knife
            hazard_index = hazard_boxes.index(hazard_box)
            label = str(classes[class_ids[hazard_index]])

            # Only consider the knife object
            if label == "knife" and not alert_triggered:
                alert_triggered, frame = display_danger_alert(frame)  # Trigger the danger alert and stop the video
                return alert_triggered, frame
    return alert_triggered, frame

# Detect hazards and monitor
def detect_hazards(frame, alert_triggered):
    height, width, _ = frame.shape
    blob = cv2.dnn.blobFromImage(frame, 0.00392, (416, 416), (0, 0, 0), True, crop=False)
    net.setInput(blob)
    outputs = net.forward(output_layers)

    class_ids = []
    confidences = []
    boxes = []

    # Process detections
    for output in outputs:
        for detection in output:
            scores = detection[5:]
            class_id = np.argmax(scores)
            confidence = scores[class_id]

            if confidence > 0.4:  # Minimum confidence threshold
                center_x = int(detection[0] * width)
                center_y = int(detection[1] * height)
                w = int(detection[2] * width)
                h = int(detection[3] * height)

                x = int(center_x - w / 2)
                y = int(center_y - h / 2)

                boxes.append([x, y, w, h])
                confidences.append(float(confidence))
                class_ids.append(class_id)

    indexes = cv2.dnn.NMSBoxes(boxes, confidences, 0.5, 0.4)

    if len(indexes) > 0:  # Check if any object is detected
        indexes = indexes.flatten()  # Extract indices from tuple

    child_boxes = []
    hazard_boxes = []

    # Track objects and draw
    for i in range(len(boxes)):
        if i in indexes:
            x, y, w, h = boxes[i]
            label = str(classes[class_ids[i]])

            # Check if 'knife' or any hazardous object is detected
            if label == "knife":  # Handle knife as a hazardous object explicitly
                color = (0, 0, 255)  # Red for hazardous objects
                hazard_boxes.append(boxes[i])
            elif label in hazard_objects:
                color = (0, 0, 255)  # Red for hazardous objects
                hazard_boxes.append(boxes[i])
            elif label == "person":  # Detect child
                color = (255, 255, 0)  # Yellow for children
                child_boxes.append(boxes[i])
            elif label in safe_objects:
                color = (0, 255, 0)  # Green for safe objects
            else:
                color = (255, 0, 0)  # Blue for other objects

            # Draw bounding box and label
            cv2.rectangle(frame, (x, y), (x + w, y + h), color, 2)
            cv2.putText(frame, label, (x, y - 10), cv2.FONT_HERSHEY_SIMPLEX, 0.9, color, 2)

    # Check proximity between child and hazardous objects (knife specifically)
    alert_triggered, frame = check_proximity_to_knife(child_boxes, hazard_boxes, class_ids, frame, alert_triggered)

    return frame, alert_triggered

# Video/Camera feed with optimizations
def start_video_feed(video_path=None, duration=60):
    # Use live camera feed if no video path is provided
    cap = cv2.VideoCapture(video_path if video_path else 0)

    if not cap.isOpened():
        print("Error: Couldn't open video source.")
        return

    start_time = time.time()  # Record the start time

    frame_skip = 8  # Skip every 2nd frame to improve speed
    frame_count = 0
    alert_triggered = False  # Initialize the alert flag

    while True:
        ret, frame = cap.read()
        if not ret:
            break

        frame_count += 1
        if frame_count % frame_skip != 0:
            continue  # Skip this frame

        frame, alert_triggered = detect_hazards(frame, alert_triggered)

        # Resize the frame to make it smaller for Jupyter display
        small_frame = cv2.resize(frame, (600, 500))  # Adjust the dimensions as per your preference

        # Display the frame in Jupyter Notebook
        _, buffer = cv2.imencode('.jpg', small_frame)
        img = PIL.Image.open(io.BytesIO(buffer))
        clear_output(wait=True)
        display(img)

        # Check if the duration has been reached or if danger alert is triggered
        if time.time() - start_time > duration or alert_triggered:
            print("Stopping video feed after danger alert.")
            break

    cap.release()
