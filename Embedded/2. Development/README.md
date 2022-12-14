# 주차장 관리 플랫폼
### `주의` : 위 개발환경은 `PC`의 `Window` 가 아닌 `Raspberry Pi 4`의 `Ubuntu 20.04`에서 진행되었습니다.
<br/>

## 작동 순서
<br/>

![squence](./image/squence.jpg)
<br/><br/>

### 1. 현황을 파악하기 위한 주차장을 촬영하여 이미지 파일로 저장합니다.
* `Result_Setting.py`의 14번째 줄 코드는 사진 촬영 함수를 호출하여 촬영한 사진을 저장합니다. 
```py
if mode == '1':
    print("\n주차장을 촬영합니다.")
    
    # 사진 촬영 함수 실행
    shoot_pic()
    
    print("주차장 촬영 완료\n")
```
<br/>

### 2. 저장된 주차장 이미지에서 주차칸 영역을 지정합니다.
* `Result_Setting.py`의 22번째 줄 코드는 저장된 주차장 사진에서 주차칸 영역의 범위를 직접 지정할 수 있는 기능을 제공합니다.
```py
# 주차칸 좌표 설정 및 수정
elif mode == '2':
    # 저장된 이미지 파일이 있는 경우
    if os.path.isfile(img_path):
        print("\n주차장 좌표 수정 및 저장을 진행합니다.")
        
        # 주차칸 좌표 데이터 파일 생성 클래스에 주차장 사진 전달
        creater = CreateCoordinateData(img_path)
        # 주차칸 좌표 데이터 파일 생성 함수 실행
        creater.SetParkingCoordinates()
        
        print("주차장 좌표 수정 및 저장을 완료했습니다.\n")
    
    # 저장된 이미지 파일이 없는 경우
    else:
        print("\n불러올 이미지가 없습니다. 주차장 촬영을 먼저 진행해주세요.\n")
```
<br/>

* `SetParkingCoordinates()` 를 가지고 있는 `CreateData.py` 의 65번째 줄 코드는 주차칸 영역을 지정한 장소의 카메라의 `고유 ID`와 `주차칸의 총 개수` 를 DB에 전송합니다.
```py
# 키보드 입력 감지
key = cv2.waitKey(1)
# 'q' 키 입력을 감지한 경우
if key == End_Key:
    getcommand = "SELECT SERIAL_ID FROM TB_PARKING_DETAIL WHERE SERIAL_ID = %s"
    get_val = (self.serial_id, )
    self.cur.execute(getcommand, get_val)
    get_list = self.cur.fetchall()
    if len(get_list) == 0:
        sendcommand = "INSERT INTO TB_PARKING_DETAIL VALUES(%s, %s, %s, %s)"
        val = (self.serial_id, 'NULL', self.id, self.enable)
        self.cur.execute(sendcommand, val)
        self.db.commit()
    else:
        sendcommand = "UPDATE TB_PARKING_DETAIL SET TOTALSPOTS = %s WHERE SERIAL_ID = %s"
        val = (self.id, self.serial_id)
        self.cur.execute(sendcommand, val)
        self.db.commit()
    
    # 무한 반복 종료
    break
```

<br/><br/>

### 3. 저장된 주차칸 영역 데이터를 불러와 주차칸에 차량 있는지 감지 시작합니다.
* `Result_Detect.py` 59번째 줄 코드는 카메라 연결 및 주차칸 데이터 불러오기, 주차칸에 차량 유무 감지를 시작합니다.
```py
# 웹에 띄우기 위한 카메라 프레임을 생성하기 위한 함수
def generate_frames():
    # 전역변수를 사용하기 위한 선언
    global cur, db, mask_list, bounds, contours, coordinates_data, serial_id, LAPLACIAN, DETECT_DELAY, job
    
    # 연결할 카메라
    cap = cv2.VideoCapture(0)
    
    # 주차칸 좌표 데이터의 값을 각각 읽기
    for p in coordinates_data:
        # 1개의 주차칸에 있는 4개 좌표들을 둘러쌀 수 있는 직사각형을 구해 저장 
        rect = cv2.boundingRect(p)
        
        # 4개 좌표 저장 리스트 복사하여 저장
        new_coordinates = p.copy()
        # 복사한 배열의 전체 row의 1번째 요소를 배열 전체 row의 1번째 요소에
        new_coordinates[:, 0] = p[:, 0] - rect[0]
        # 복사한 배열의 전체 row의 2번째 요소를 배열 전체 row의 2번째 요소에
        new_coordinates[:, 1] = p[:, 1] - rect[1]
        
        # 검출한 윤곽선을 리스트에 저장
        contours.append(p)
        # 검출한 직사각형을 리스트에 저장
        bounds.append(rect)
        
        # 검출한 윤곽선을 화면에 그리기
        mask = cv2.drawContours(np.zeros((rect[3],rect[2]), dtype=np.uint8), [new_coordinates], contourIdx = -1, color = 255, thickness = -1, lineType=cv2.LINE_8)
        # 255를 초과하는 mask 값일 경우 255로 고정
        mask = mask == 255
        # 검출한 mask 값들을 mask 리스트에 저장
        mask_list.append(mask)
    
    # 주차칸 상태를 저장할 리스트
    statuses = [False] * len(coordinates_data)
    # 주차칸에서 차량을 탐지하기 시작해 카운트한 시간 리스트
    times = [None] * len(coordinates_data)
    
    # 주기적으로 DB 서버에 정보를 보내는 함수를 실행
    job = schedule.every().minute.at(":30").do(send_log, statuses)
    
    # 무한 반복
    while True:        
        # 주기적으로 데이터를 전송하는 함수를 실행시키기 시작
        schedule.run_pending()
        
        # 카메라 영상의 프레임 읽기
        ret, frame = cap.read()
        
        # 프레임에 가우시안 필터 사용
        blurred = cv2.GaussianBlur(frame, (5, 5), 3)
        # 가우시안 필터링한 프레임을 흑백으로 변환
        grayed = cv2.cvtColor(blurred, cv2.COLOR_BGR2GRAY)
        
        # 카메라 영상에서 사용할 1초 시간 정의
        position_in_seconds = cap.get(cv2.CAP_PROP_POS_MSEC) / 1000.0
        
        # 주차칸 좌표 데이터를 순서, 4개의 좌표 데이터로 리스트에서 나누기
        for index, c in enumerate(coordinates_data):
            # 특정 순서 주차칸의 상태를 함수에서 계산 반환하여 저장
            status = __apply(grayed, index, c)
            
            # 탐지 시간 기록이 있고 주차칸의 이전 상태와 현재 상태가 같은 경우
            if times[index] is not None and same_status(statuses, index, status):
                # 탐지 시간에 변화없음을 인지
                times[index] = None
                continue
            
            # 탐지 시간 기록이 있고 주차칸의 이전 상태와 현재 상태가 같지 않은 경우
            if times[index] is not None and status_changed(statuses, index, status):
                # 탐지되는 시간이 기준으로 정한 탐지 시간 이상인 경우
                if position_in_seconds - times[index] >= DETECT_DELAY:
                    # 주차칸 상태를 현재 상태로 저장
                    statuses[index] = status
                    # 탐지 시간 기록 초기화
                    times[index] = None
                continue
            
            # 탐지 시간 기록이 없고 주차칸의 이전 상태와 현재 상태가 같지 않은 경우
            if times[index] is None and status_changed(statuses, index, status):
                # 탐지 시간 카운트 시작
                times[index] = position_in_seconds
        
        # 주차칸 좌표 데이터를 순서, 4개의 좌표 데티어로 리스트에서 나누기
        for index, c in enumerate(coordinates_data):
            # 주차칸 현재 상태가 True면 녹색(주차 가능) / False면 빨간색(주차 불가능)으로 구분하기
            color = green_color if statuses[index] else red_color
            # 주차칸 가장자리, 가운데에 주차칸 번호를 그리기
            draw_contours(frame, c, str(index + 1), white_color, color)
                
        key = cv2.waitKey(1)
        # 'q' 키를 눌렀을 경우
        if key == ord('q'):
            # 무한 반복 종료
            break
        
        # 프레임을 jpg 파일로 인코딩하기 (배열로 변환해 buffer 저장)
        result, buffer = cv2.imencode('.jpg',frame)
        # 배열 데이터를 바이트 문자열로 변환
        frame = buffer.tobytes()
        # Generate 생성
        yield (b'--frame\r\n'
            b'Content-Type: image/jpeg\r\n\r\n' + frame + b'\r\n')

    # DB 서버와 연결 해제
    cur.close()
    db.close()
    # 모든 윈도우창 닫기
    cv2.destroyAllWindows()
```
<br/>

* `Result_Detect.py` 168번째 줄 코드는 수집한 주차장 현황 정보를 일정 시간을 기준으로 DB에 전송하는 함수입니다.
* 이 함수는 97번째 줄 코드 `job = schedule.every().minute.at(":30").do(send_log, statuses)` 를 통해 함수가 호출될 주기를 설정합니다.
* 그리고 위 설정은 102번째 줄 코드 `schedule.run_pending()` 으로 매 주기마다 실행하도록 합니다.
```py
# DB 서버에 데이터를 전송하는 함수
def send_log(statuses_list):
    db = mysql.connector.connect(host='your_DB_address', port='your_DB_port', user='your_user', password='your_DB_password!', database='your_DB_name', auth_plugin='mysql_native_password')
    cur = db.cursor()
    
    # 전역변수를 사용하기 위한 선언
    global mask_list, bounds, contours, coordinates_data, serial_id, LAPLACIAN, DETECT_DELAY

    # 주차칸 현재 상태 리스트에서 순서, 상태로 나누어 읽어 주차 가능한 자리(True)인 경우 해당 순서 주차칸 번호를 리스트에 저장
    enable_list = [index + 1 for index, value in enumerate(statuses_list) if value != False]
    # 저장한 리스트를 문자열로 변환
    enable = ','.join(list(map(str, enable_list)))
    
    # 주차칸 현재 상태 리스트에서 순서, 상태로 나누어 읽어 주차 불가능한 자리(False)인 경우 해당 순서 주차칸 번호를 리스트에 저장
    disable_list = [index + 1 for index, value in enumerate(statuses_list) if value == False]
    # 저장한 리스트를 문자열로 변환
    disable = ','.join(list(map(str, disable_list)))
    
    # DB 서버에 기록 데이터를 보내 위한 mysql 명령어
    sendcommand = "CALL PD_LOG_INSERT(%s, %s, %s, %s, %s)"
    # sendcommand = "INSERT INTO TB_PARKING_LOG (SERIAL_ID, TOTAL, ENABLE, ENABLELIST, OCUPIEDLIST) VALUES (%s, %s, %s, %s, %s)"
    # DB 서버에 보낼 데이터들
    val = (serial_id, len(statuses_list), len(enable_list) , enable, disable)
    # DB 서버에 전송 시작
    cur.execute(sendcommand, val)
    db.commit()
```
<br/>

### 4. 카메라로 촬영 중인 영상을 웹 스트리밍 할 수 있도록 만들어줍니다.
* `Result_Detect.py` 233번째 줄 코드는 웹 스트리밍이 가능하도록 만들어줍니다. 웹 스트리밍할 카메라 촬영중인 영상을 담을 함수를 생성하고 `index.html` 에서 함수를 호출하면 해당 영상을 웹에서 확인할 수 있습니다.
```py
# 서버 url 호출을 위한 주소 '/video'의 정의
# 웹에 카메라 영상을 띄우기 위한 함수
@app.route('/video')
def video():    
    schedule.cancel_job(job)
    return Response(generate_frames(),mimetype='multipart/x-mixed-replace; boundary=frame')

# 서버 url 호출을 위한 주소 '/'의 정의
# 웹 화면을 구성하는 html 파일 호출
@app.route('/')
def index():
    schedule.cancel_job(job)
    return render_template('index.html')

if __name__ == '__main__':
    # debug 가능, 같은 네트워크가 연결되있는 경우 어느 컴퓨터에서 접속 가능
    app.run(host='0.0.0.0')
``` 