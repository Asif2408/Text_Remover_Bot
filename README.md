import boto3
import io
import base64
import json
import pytesseract
import subprocess
import sys
import string
import pickle
import opencv

def main():
    # AWS credentials provider chain
    session = boto3.Session()
    
    # Get access to the s3 bucket and object key
    svc = session.client('s3', region_name='us-east-1')
    obj = {'Key': 'your_object_key_here'}
    response = svc.list_objects_v2(Bucket='your_bucket_name_here', Prefix='images/')
    objects = response['Contents']
    
    texts = []
    img_paths = []
    for object in objects:
        if '.jpg' not in object['Key'].split('/')[-1]:
            continue
        
        # Download file to local storage
        out = open(f"image_{object['Key']}.jpg", "wb")
        obj['BodyIO']['Value'].readinto(out)
        out.close()
        
        # Read image from file path and apply image processing
        img = cv2.imread(f"{sys.prefix}/Downloads/image_{object['Key']}")

        # Apply preprocessing including resizing, converting RGB to grayscale and thresholding
        img = cv2.resize(img, None, fx=0.5, fy=0.5)
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

        blurred = cv2.GaussianBlur(gray, (31, 31), 0)
        _, thresh = cv2.threshold(blurred, 157, 2555, cv2.THRESH_BINARY_INV + cv2.THRESH_OTSU)

        # Remove noisy pixels (including text)
        contours, _ = cv2.findContours(thresh, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
        cnt = max(contours, key=(lambda x: cv2.contourArea(x)))
        mask = np.zeros((img.shape)[::-1], dtype=np.uint8)
        cv2.fillPoly(mask, [(tuple(map(int(i * 3) for i in cnt[0].tolist())) for i in range(len(cnt[0]))]), green=[0, 0, 0])

        # extract text and save output image
        txt = pytesseract.image_to_string(thresh, config='--psm 1 --deskew 0.7 -c tessedit_char_whitelist=0123456789ABCDEFGHIJKLMNPQRSTUVWXYZ')
        text_files = {"text": txt}
        img_path = f"./image_{o
