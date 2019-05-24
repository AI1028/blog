```python
#!/usr/bin/env python
#coding=utf8

# --------------------------------------------------------
# Tensorflow Faster R-CNN
# Licensed under The MIT License [see LICENSE for details]
# Written by Xinlei Chen, based on code from Ross Girshick
# --------------------------------------------------------

"""
Demo script showing detections in sample images.

See README.md for installation instructions before running.
"""
from __future__ import absolute_import
from __future__ import division
from __future__ import print_function

import _init_paths
from model.config import cfg
from model.test import im_detect
from model.nms_wrapper import nms

from utils.timer import Timer
import tensorflow as tf
import matplotlib.pyplot as plt
import numpy as np
import os, cv2
import argparse

from flask import request, Flask
from flask import jsonify
import time
import os
import json
import io
import base64
from PIL import Image
import zlib
import copy

from nets.vgg16 import vgg16
from nets.resnet_v1 import resnetv1

import Face

from gevent import monkey
from gevent.pywsgi import WSGIServer

CLASSES = ('__background__', 
           'aeroplane', 'bicycle', 'bird', 'boat',
           'bottle', 'bus', 'car', 'cat', 'chair',
           'cow', 'diningtable', 'dog', 'horse',
           'motorbike', 'person', 'pottedplant',
           'sheep', 'sofa', 'train', 'tvmonitor', 'face')

color = ((255, 255, 255), (211, 85, 186), (100, 149, 237), (123, 104, 238), 
        (0, 191, 255), (0, 255, 255), (0, 250, 154), (238, 221, 130), 
        (205, 92, 92), (255, 105, 180), (138, 43, 226), (131, 111, 255), 
        (30, 144, 255), (0, 245, 255), (144, 238, 144), (154, 255, 154), 
        (0, 255, 0), (255, 246, 143), (255, 255, 0), (255, 106, 106), 
        (255, 255, 0))

NETS = {'vgg16': ('vgg16_faster_rcnn_iter_7000.ckpt',),'res101': ('res101_faster_rcnn_iter_110000.ckpt',)}
DATASETS= {'pascal_voc': ('voc_2007_trainval',),'pascal_voc_0712': ('voc_2007_trainval+voc_2012_trainval',)}

def vis_detections(im, class_name, dets, detection_result, thresh=0.5):
    """Draw detected bounding boxes."""
    inds = np.where(dets[:, -1] >= thresh)[0]
    if len(inds) == 0:
        return
    detection_result[class_name + '_num'] = len(inds)
    for i in inds:
        bbox = dets[i, :4]
        score = dets[i, -1] 
        tmp = {'location': bbox.tolist(), 'score': score.tolist()} 
        detection_result[class_name + '_list'].append(tmp)

def demo(sess, net, frame, detection_result):
    global frameRate
    timer = Timer()
    im = frame
    timer.tic()
    scores, boxes = im_detect(sess, net, im)
    timer.toc()
    frameRate = 1.0 / timer.total_time
    CONF_THRESH = 0.8
    NMS_THRESH = 0.3
    for cls_ind, cls in enumerate(CLASSES[1:21]):
        cls_ind += 1
        cls_boxes = boxes[:, 4*cls_ind:4*(cls_ind + 1)]
        cls_scores = scores[:, cls_ind]
        dets = np.hstack((cls_boxes,
                          cls_scores[:, np.newaxis])).astype(np.float32)
        keep = nms(dets, NMS_THRESH)
        dets = dets[keep, :]
        vis_detections(im, cls, dets, detection_result, thresh=CONF_THRESH)

'''
def parse_args():
    """Parse input arguments."""
    parser = argparse.ArgumentParser(description='Tensorflow Faster R-CNN demo')
    parser.add_argument('--net', dest='demo_net', help='Network to use [vgg16 res101]',
                        choices=NETS.keys(), default='res101')
    parser.add_argument('--dataset', dest='dataset', help='Trained dataset [pascal_voc pascal_voc_0712]',
                        choices=DATASETS.keys(), default='pascal_voc_0712')
    args = parser.parse_args()

    return args
'''


cfg.TEST.HAS_RPN = True
#args = parse_args()

    
demonet = 'res101'#args.demo_net
dataset = 'pascal_voc_0712'#args.dataset
tfmodel = os.path.join('../output', demonet, DATASETS[dataset][0], 'default', NETS[demonet][0])


if not os.path.isfile(tfmodel + '.meta'):
    raise IOError(('{:s} not found.\nDid you download the proper networks from '
                   'our server and place them properly?').format(tfmodel + '.meta'))

    
tfconfig = tf.ConfigProto(allow_soft_placement=True)
tfconfig.gpu_options.allow_growth=True

    
sess = tf.Session(config=tfconfig)
    
if demonet == 'vgg16':
    net = vgg16()
elif demonet == 'res101':
    net = resnetv1(num_layers=101)
else:
    raise NotImplementedError
net.create_architecture("TEST", 21, tag='default', anchor_scales=[8, 16, 32])
saver = tf.train.Saver()
saver.restore(sess, tfmodel)

# monkey.patch_all()

app = Flask(__name__)
@app.route("/", methods=['POST'])

def detection():
    detection_result = {'aeroplane_num': 0, 'aeroplane_list': [], 'bicycle_num': 0, 'bicycle_list': [], 'bird_num': 0, 'bird_list': [],'boat_num': 0, 'boat_list': [],'bottle_num': 0, 'bottle_list': [],'bus_num': 0, 'bus_list': [],'car_num': 0, 'car_list': [],'cat_num': 0, 'cat_list': [],'chair_num': 0, 'chair_list': [],'cow_num': 0, 'cow_list': [],'diningtable_num': 0, 'diningtable_list': [],'dog_num': 0, 'dog_list': [],'horse_num': 0, 'horse_list': [],'motorbike_num': 0, 'motorbike_list': [],'person_num': 0, 'person_list': [],'pottedplant_num': 0, 'pottedplant_list': [],'sheep_num': 0, 'sheep_list': [],'sofa_num': 0, 'sofa_list': [],'train_num': 0, 'train_list': [],'tvmonitor_num': 0, 'tvmonitor_list': [], 'face_num': 0, 'face_list': []}

    for x in CLASSES[1:]:
        detection_result[x + '_num'] = 0
        detection_result[x + '_list'] = []

    if request.is_json:
        res = request.json
        frame = res['image']
        frame = Image.frombytes('RGB', res['image_size'], zlib.decompress(eval(frame)))
        frame = np.array(frame, dtype=np.float32)
        frame = cv2.resize(frame, None, fx=1/res['ratio'], fy=1/res['ratio'])


        frame = cv2.GaussianBlur(frame, (5, 5), 0)  # lv bo
        kernel = np.array([[0, -1, 0], [-1, 5, -1], [0, -1, 0]], np.float32)
        frame = cv2.filter2D(frame, -1, kernel=kernel)  # rui hua



        demo(sess, net, frame, detection_result)
        detection_result['face_num'] = 0
        detection_result['face_list'] = []

        frame = frame.astype(np.uint8) 
        faces = Face.face(frame)
        detection_result['face_num'] = len(faces[0])
        for i in range(len(faces[0])):
            tmp = {'location': faces[0][i], 'name': faces[1][i]}
            detection_result['face_list'].append(tmp)
        # print(detection_result)
        return allow_origin(jsonify(detection_result))

    else:
        form_data = request.form
        img_base64 = form_data.get('image', '')
        nparray = np.fromstring(base64.b64decode(img_base64), np.uint8)
        frame = cv2.imdecode(nparray, cv2.IMREAD_COLOR)
        demo(sess, net, frame, detection_result)

        detection_result['face_num'] = 0
        detection_result['face_list'] = []

        frame = frame.astype(np.uint8) 

        faces = Face.face(frame)
        detection_result['face_num'] = len(faces[0])
        for i in range(len(faces[0])):
            tmp = {'location': faces[0][i], 'name': faces[1][i]}
            detection_result['face_list'].append(tmp)

        response = detection_result
        copy_response = copy.deepcopy(detection_result)
        
        for z, x in enumerate(CLASSES[1:21]):
            object_list = response[x + '_list']
            for y in object_list:
                y['location'] = np.array(y['location'], dtype=np.float32)
                top, right, bottom, left, score = y['location'][0], y['location'][1], y['location'][2], y['location'][3], y['score']
                cv2.rectangle(frame, (top, right), (bottom, left), color[z], 2)
                cv2.putText(frame, '{:s}'.format(x), (int(top), int(right - 5)), cv2.FONT_HERSHEY_DUPLEX, 0.62, color[z])
                cv2.putText(frame, '            {:.3f}'.format(y['score']), (int(top), int(right - 5)), cv2.FONT_HERSHEY_DUPLEX, 0.5, color[z])
                

        for face in response['face_list']:
            top, right, bottom, left, name = face['location'][0], face['location'][1], face['location'][2], face['location'][3], face['name']
            cv2.rectangle(frame, (left, top), (right, bottom), color[20], 2)
            cv2.putText(frame, name, (left, bottom + 15), cv2.FONT_HERSHEY_DUPLEX, 0.5, color[20])

        image = cv2.imencode(".jpeg", frame)[1]
        image_base64 = str(base64.b64encode(image))[2:-1]
        data={"image":image_base64, "Response":copy_response}
        return allow_origin(jsonify(data))


def allow_origin(response):
    response.headers['Access-Control-Allow-Origin']='*'
    return response


if __name__ == "__main__":
    # app.run("192.168.3.11", port=7000)
    app.run(host='127.0.0.1', port='3000', debug=False, threaded=True)
    # app.run("172.16.60.62", port=7000)
    # WSGIServer(('127.0.0.1', 2000), app).serve_forever()



```