# Drawing-recognition

## 목차

- [요약](#요약)
- [데이터셋 만들기](#데이터셋-만들기)
- [학습](#학습)  
- [Kinect를 이용한 손그림 인식](#Kinect를-이용한-손그림-인식) 

</br>
</br>
</br>

---
## 요약

- 사람이 허공에 대고 그린 그림을 실시간으로 인식하는 딥러닝 모델을 구현
- 데이터셋을 구축하기 위해 google에서 오픈한 데이터셋 quick draw를 활용
    - 이미지 파일이 아닌 json 파일로 구성되어 있어 이를 이미지로 변환하는 작업

</br>
</br>
</br>

---
## 데이터셋 만들기

### 1. https://github.com/googlecreativelab/quickdraw-dataset 참고
![quickdraw_logo](https://user-images.githubusercontent.com/42198637/88623215-71938880-d0df-11ea-8ec4-092e8fd24f53.PNG)


### 2. raw data download

- 위 깃허브의 ReadMe.md에서 에서 Get the data 항목에 가면 데이터셋 정보 파일(.ndjson, .npy, ...)들이 저장되어있는 구글 클라우드 링크가 있다.
    - https://console.cloud.google.com/storage/browser/quickdraw_dataset?pli=1

        ![quickdraw_cloud](https://user-images.githubusercontent.com/42198637/88623339-bae3d800-d0df-11ea-993a-a96f4d0fdaf9.PNG)

- Google Cloud Console을 다운받은 후 실행하면 파일을 일괄로 다운받을 수 있다.(구글 계정 필요)
    - https://cloud.google.com/sdk/docs/quickstart-windows?hl=ko
    - 아래 명령어를 입력하여 일괄 다운
         ```
        gsutil cp [구글 클라우드 경로] [내 PC 경로]
        gsutil cp gs://quickdraw_dataset/full/numpy_bitmap/*.npy D:\yjlee\data\quickdraw
        ```
    
- 파일 항목은 다음과 같다.
    |항목|확장자|내용|
    |:---|:---|:---|
    |binary|.bin||
    |numpy_bitmap|.npy||
    |raw|.ndjson|그림 좌표 정보|
    |simplified|.ndjson|raw 용량 줄인거|

- 다운받은 파일 중 그림 좌표가 저장되어있는 raw만 가져다 썼다.

### 3. .ndjson to .json 변환

- openFrameworks에서 raw 파일을 다루기 위해서 ndjson 파일을 json으로 변환할 필요가 있다.
- 코드는 [ndjson-to-json](https://github.com/dog0029/ndjson-to-json) 에 있다.


### 4. 이미지 생성
- 코드는 \\1.Quick Draw\\코드\\그림 데이터셋 구축\\2.json_to_image 에 있다.
- raw 파일의 그림 양식은 다음과 같다.
    ```javascript
    { 
        "key_id":"5891796615823360",
        "word":"nose",
        "countrycode":"AE",
        "timestamp":"2017-03-01 20:41:36.70725 UTC",
        "recognized":true,
        "drawing":[[[129,128,129,129,130,130,131,132,132,133,133,133,133,...]]]
    }
    ```
- 저기서 drawing 항목에 그림 좌표가 저장되어 있다. drawing 포멧은 다음과 같다.
    ```javascript
    [ 
        [  // First stroke 
            [x0, x1, x2, x3, ...],
            [y0, y1, y2, y3, ...],
            [t0, t1, t2, t3, ...]
        ],
        [  // Second stroke
            [x0, x1, x2, x3, ...],
            [y0, y1, y2, y3, ...],
            [t0, t1, t2, t3, ...]
        ],
        ... // Additional strokes
    ]
    ```
    - 첫번째 대괄호는 그림 단위, 두번째 대괄호는 선단위, 세번째 대괄호는 좌표 단위이다.
    - x는 x좌표 모음, y는 y좌표 모음으로 각 1대1 매칭하면 된다.
    - t는 msec 단위로 시작점을 알려준다고 하는데 아직 잘 모르겠다.
    
- openFramworks에서 json 파일을 다룰 수 있다.
    - https://openframeworks.cc/download/ 에서 VS2017로 다운
    - examples\input_output\jsonExample 에서 확인 가능
    - json 예제 소스 코드
    ```cpp
    ofJson js;
    ofFile file("drawing.json");
	if(file.exists())
	{
	    file >> js;
	    for(auto & stroke: js)
	    {
	        if(!stroke.empty())
	        {
                // 다음 점으로 이동
		        path.moveTo(stroke[0]["x"], stroke[0]["y"]);
		        for(auto & p: stroke)
		        {
                    // 이전 점에서 해당 점까지 직선을 그림
		        	path.lineTo(p["x"], p["y"]);
		        }
		    }
	    }
    }
    ```
    - ofJson에 파일을 넣으면 for문으로 json 괄호 안에 접근이 가능하다.
    - moveTo로 다음 점으로 이동하고 lineTo는 다음 점까지 직선을 그린다.
        - 그림을 그릴 때 선의 시작점은 moveTo로 하고 나머지(해당 선의 끝점까지)는 lineTo를 적용한다.
        - 그리고 다음 선의 시작점은 moveTo로 하며 반복한다.

- 생성된 이미지 예시

    ![quickdraw_apple](https://user-images.githubusercontent.com/42198637/88624561-2c248a80-d0e2-11ea-8a5f-8ab01d6929a6.PNG)


### 5. 데이터셋 구축

- yolo 학습 데이터셋 
1. json 파일에 저장된 좌표 정보를 가지고 그림 이미지 만들기
2. 각 이미지에 대한 바운딩 박스 txt 파일 만들기(1대1 대응)
3. train 데이터와 valid 데이터로 구분하기(valid는 클래스 별 25000장의 이미지로 구성)
4. 이미지 경로 리스트 txt 파일 만들기

> 인수인계 백업에는 용량 문제로 인해 .ndjson 파일만 넣음

</br>
</br>
</br>

---
## 학습

### 1. yolov3 학습

- <del>배경제거된 이미지로 학습</del> 
    - loss가 0.7 밑으로 떨어지지 않음
    - 테스트 하였을 때 전혀 클래스 분류가 되지 않음
- 흰 배경인 이미지로 학습
    - loss는 조금씩 떨어지고 있음(iter 228600 : loss 0.1715)
    - 테스트 결과 이미지
    ![result](https://user-images.githubusercontent.com/42198637/91007546-38cfcc00-e617-11ea-98d3-062caa875509.png)

</br>
</br>
</br>

---
## Kinect를 이용한 손그림 인식

### 1. Kinect 연동 및 손인식 구현

- Kinect를 연동하여 손을 인식하고 손의 동작에 맞춰 그림을 그린다.
    - Kinect 내부적으로 제공하는 손동작과 작업 내용은 다음과 같다.
        - 가위(V 표시한 손) : 손으로 V 표시를 하여 그려진 그림을 모두 지운다.
        - 바위(주먹쥔 손) : 손을 주먹 쥔 상태에서 손을 움직여 그림을 그린다.
        - 보(쫙 핀 손) : 손을 쫙 핀 상태에서 그리기를 멈추고 손을 다른 위치로 이동시킨다.
    - 그림을 그리는 모듈은 여러가지가 있다.
        - 손이 있는 좌표에 점찍기
        ```cpp
        // ofApp::draw()
        ofSetColor(0, 0, 0); // 색 지정
		ofSphere(x, y, 10);
		ofSetColor(255, 255, 255); // 초기화
        ```
        - 직선 그리기(moveTo, lineTo)
        ```cpp
        // ofApp::setup()
        ofColor color(0, 0, 0);
	    path.setStrokeColor(color);
	    path.setFilled(false);
	    path.setStrokeWidth(100);

        // ofApp::draw()
        path.draw();

        path.moveTo(x1, y1);  // 다음 점으로 이동
        path.lineTo(x2, y2);  // 이전 점에서 인자값(x2, y2)까지 직선
        ```
        - 직선 그리기(ofLine)
        ```cpp
        // ofApp::setup()
        ofSetLineWidth(100);

        // ofApp::draw()
        ofSetColor(0, 0, 0);
        ofLine(x1, y1, x2, y2); // (x1, y1) -> (x2, y2) 직선그리기
        ofSetColor(255, 255, 255);
        ```

- 다 그려진 그림을 이미지 파일로 저장한다.(모델에 입력하기 위한 변수에 저장)
    - 이미지를 저장할 때 kinect 화면은 제외해야 하는데 이를 위해 layer를 분리한다.
    - layer 분할은 ofFbo를 활용하였다.
        ```cpp
        // ofApp::setup()
        DrawingLayer.allocate(COLOR_WIDTH, COLOR_HEIGHT);

        // ofApp::draw()
        DrawingLayer.begin();
        /* drawing... */
        DrawingLayer.end();

        // 이후 layer에서 그려진 그림 픽셀값을 ofImage에 적용
        ```
    - 이후 그림 이미지 크기만큼만 저장하기 위해서 이미지를 crop 한다.
        - 참고 : https://forum.openframeworks.cc/t/video-grabber-crop-to-image-issue/26040/6
        ```cpp
        // 각 그림 좌표를 저장하고 있다가 save할 때 그림의 최소, 최대 좌표를 구한다.
        // grabscreen 함수로 하지 않는다.
		save_image.crop(min_x, min_y, width, height);
        ```
    - yolo 모델에 입력하기 위해 그림 데이터의 배경(흰색)을 넣어준 뒤 그림 이미지를 저장한다.
        - 먼저 setImageType 함수를 통해 이미지의 타입을 변경하고(ALPHA 없도록)
            - 참고 : https://forum.openframeworks.cc/t/set-white-pixels-to-transparent/23792/2
        -  이후 opencv를 통해 이미지를 반전시켰다.
            - 참고 : https://forum.openframeworks.cc/t/invert-image-with-ofxtoggle-doesnt-work/31202

        
### 2. yolov3 네트워크 추가

- opencv는 openframeworks에 내장되어있는 라이브러리를 사용하고 yolo는 darknet에서 빌드된 dll을 사용하였다.
- 오류
    1. yolo_v2_class.hpp
        - std::isnan 에서 식별자를 찾을 수 없다는 오류 -> std::를 지웠다
        - 아마 namespace std;가 되어있어서 지워야한다는듯
        - 참고 : https://stackoverrun.com/ko/q/5760062
    2. opencv 충돌
        - openframeworks 내장 opencv와 yolo에서 사용할 라이브러리 버전이 다르다.
        - 포함, 라이브러리 디렉터리에 yolo opencv버전(3.4.1) 경로를 넣어주고
        - 추가 종속성에 기존(openframeworks)의 opencv lib파일은 모두 지운 뒤 yolo lib을 입력하였다.
    3. 이미지 입력
        - yolo에 입력할 이미지를 cv::imread를 통해 load할 수 없고 ofImage로 load한 후 cv::Mat 형식으로 변환하였다.
        ```cpp
        ofImage tmps;
	    tmps.load("image/calculater.PNG");
        cv::Mat images = ofxCv::toCv(tmps);
        ```

### 3. 인식 결과 표출

- yolov3를 통해 인식된 결과를 화면에 출력한다.
- 프로그램 흐름도
![quickdraw_flow](https://user-images.githubusercontent.com/42198637/91810470-e28f0880-ec68-11ea-805b-3dba844500c5.PNG)
