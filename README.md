# main.py
import cv2
import time

# ================= 配置区 =================
DEBRIS_AREA_THRESHOLD = 3000  # 污染识别阈值
CAMERA_INDEX = 0              # 摄像头编号

# 硬件引脚（这里先使用模拟引脚，真实硬件时请修改）
MOTOR_IN1 = 17
MOTOR_IN2 = 18
BRUSH_PIN = 23
# =========================================

class Motor:
    """电机控制模拟"""
    def __init__(self, in1, in2):
        self.in1 = in1
        self.in2 = in2
        self.running = False

    def forward(self):
        self.running = True
        print("电机：前进")

    def stop(self):
        self.running = False
        print("电机：停止")

class Brush:
    """滚刷控制模拟"""
    def __init__(self, pin):
        self.pin = pin
        self.on = False

    def start(self):
        self.on = True
        print("滚刷：开启")

    def stop(self):
        self.on = False
        print("滚刷：关闭")

class Camera:
    """摄像头操作"""
    def __init__(self, index, width=640, height=480):
        self.cap = cv2.VideoCapture(index)
        self.cap.set(cv2.CAP_PROP_FRAME_WIDTH, width)
        self.cap.set(cv2.CAP_PROP_FRAME_HEIGHT, height)

    def read(self):
        ret, frame = self.cap.read()
        if not ret:
            return None
        return frame

    def release(self):
        self.cap.release()
        cv2.destroyAllWindows()

class DebrisDetector:
    """落叶/污染源识别器"""
    def detect(self, frame):
        hsv = cv2.cvtColor(frame, cv2.COLOR_BGR2HSV)
        
        # 识别绿色落叶
        leaf_mask = cv2.inRange(hsv, (35, 40, 40), (85, 255, 255))
        # 识别深色污渍/鸟粪
        dark_mask = cv2.inRange(hsv, (0, 0, 0), (180, 255, 80))
        
        # 合并遮罩层
        debris_mask = cv2.bitwise_or(leaf_mask, dark_mask)
        debris_area = int(cv2.countNonZero(debris_mask))
        
        return debris_area, debris_mask

class PVCleanRobot:
    """主机器人控制逻辑"""
    def __init__(self):
        self.camera = Camera(CAMERA_INDEX)
        self.detector = DebrisDetector()
        self.motor = Motor(MOTOR_IN1, MOTOR_IN2)
        self.brush = Brush(BRUSH_PIN)
        self.running = True

    def run(self):
        print("光伏清扫机器人启动...")

        while self.running:
            frame = self.camera.read()
            if frame is None:
                print("未获取到摄像头画面")
                time.sleep(0.5)
                continue

            debris_area, mask = self.detector.detect(frame)

            if debris_area > DEBRIS_AREA_THRESHOLD:
                print(f"检测到污染源，污染面积={debris_area}")
                self.motor.forward()
                self.brush.start()
            else:
                print(f"表面较干净，污染面积={debris_area}")
                self.motor.stop()
                self.brush.stop()

            # 实时显示识别遮罩画面
            cv2.imshow("Debris Detection", mask)
            if cv2.waitKey(1) & 0xFF == ord('q'):
                self.running = False

        self.motor.stop()
        self.brush.stop()
        self.camera.release()
        print("机器人停止，程序退出。")

if __name__ == "__main__":
    robot = PVCleanRobot()
    robot.run()
