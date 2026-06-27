+++
date = '2026-06-08T22:46:12+07:00'
draft = false
title = 'Debug ยังไงดี ? เราไม่มี shell ใน container! วิธีใช้ kubectl debug ทะลวงเข้า Distroless Container ใน Kubernetes'
+++
![distorless Debugging](/images/yamu_jay-computer-9056546.jpg)


ในการสร้าง Container Image ระดับ Production มาตรฐานที่เหล่า DevOps และ System Architect มักจะเลือกใช้กันคือการทำ Distroless Image หรือการใช้ FROM scratch เนื่องจากมันมอบข้อได้เปรียบที่สำคัญ 3 ประการให้กับระบบ:

🚀- Performance & Speed (ขนาดเล็ก โหลดไว): ตัด Component ของระบบปฏิบัติการที่พะรุงพะรังออกไป ทำให้ Image มีขนาดเล็กจิ๋ว ดึง (Pull) จาก Registry ได้ไว และ Start Container ได้รวดเร็ว

🛡️ - Security (ลดช่องโหว่): ลด Attack Surface หรือพื้นที่การโจมตีจากภายนอกได้อย่างชะงัด เพราะไม่มี Shell (/bin/bash หรือ /bin/sh) และไม่มี Utilities เครื่องมือแฮกใดๆ ติดมาใน Image เลย

   - Stability (ความเสถียรขั้นสุด): ตัดความซับซ้อนทิ้ง มีเฉพาะ Application Binary และ Dependencies ที่จำเป็นต่อการรันโปรแกรมเท่านั้น

### Pain Point: ปลอดภัยสุดขีด แต่ Debug ชีวิตเปลี่ยน
ความปลอดภัยที่ได้มา แลกมากับความเจ็บปวดตอนที่ระบบมีปัญหาบน Production ครับ เพราะโดยปกติเวลาเราต้องการเข้าไปดู State ภายใน Pod เรามักจะใช้ท่ามาตรฐานคือ:

```bash
kubectl exec -it <pod-name> -c <container-name> -- /bin/bash
```

แต่เมื่อ Base Image ของเราเป็น Distroless ซึ่ง ไม่มี Shell คำสั่ง kubectl exec จะพ่น Error กลับมาทันที ทำให้เราตาบอด ไม่สามารถเข้าไปรันคำสั่งพื้นฐานอย่าง top, netstat, หรือ curl เพื่อหาสาเหตุของปัญหาได้เลย

### The Solution: ทะลวงผ่าน Ephemeral Container ด้วย kubectl debug
ตั้งแต่ Kubernetes v1.23 เป็นต้นมา ได้มีการเปิดตัวฟีเจอร์ใหม่ที่เข้ามาพลิกเกมเรื่องนี้ นั่นคือ Ephemeral Containers แนวคิดของมันคือการสร้าง Container ชั่วคราว (เช่น อัดเครื่องมือ Debug เข้าไปเต็มพิกัดอย่าง busybox หรือ netshoot) แล้วนำไปแปะติด (Attach) เป็น Sidecar เข้ากับ Pod ที่กำลังรันอยู่ ทำให้เราสามารถเข้าไปแชร์ Process Namespace และจัดการกับ Container เป้าหมายได้โดยไม่ต้องแก้ Image เดิมเลย

### Hands-on: วิธีการใช้งานจริง
เพื่อให้เห็นภาพการทำงาน เรามาลองจำลองสถานการณ์กันครับ

- Step 1: สร้าง Target Pod ที่ต้องการ (สมมติว่าเป็น Nginx)

```bash
kubectl run server --image nginx
```
- Step 2: สั่ง Attach Debugger Container
เราจะใช้คำสั่ง kubectl debug เพื่อยัด Image busybox เข้าไปแชร์สภาพแวดล้อมกับ Container ที่ชื่อ server

```bash
kubectl debug server -c debugger -it --image=busybox --target=server /#
```

- Step 3: ตรวจสอบ Process ภายใน
เมื่อ Shell เปิดขึ้นมา (ซึ่งเป็น Shell ของ Busybox แต่แชร์ Process Namespace กับ Nginx) เราลองใช้คำสั่ง top เพื่อดูการทำงาน:

```bash
/# top

Mem: 7482596K used, 8907768K free, 37104K shrd, 456252K buff, 3180508K cached
CPU:  2.5% usr  0.7% sys  0.0% nic 96.4% idle  0.2% io  0.0% irq  0.0% sirq
Load average: 0.30 0.20 0.16 2/796 40

  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
   29     1 101      S    11868  0.0   0  0.0 nginx: worker process
   30     1 101      S    11868  0.0   1  0.0 nginx: worker process
   31     1 101      S    11868  0.0   2  0.0 nginx: worker process
   32     1 101      S    11868  0.0   3  0.0 nginx: worker process
    1     0 root     S    11404  0.0   3  0.0 nginx: master process nginx -g daemon off;
   33     0 root     S     4404  0.0   0  0.0 sh
   40    33 root     R     4404  0.0   1  0.0 top
```

จะเห็นได้ชัดเจนว่าเราสามารถมองเห็น Process ของ nginx (PID 1 และผองเพื่อน) ได้อย่างครบถ้วนผ่าน Ephemeral Container ตัวนี้  

- Step 4: ดูสิ่งที่เกิดขึ้นเบื้องหลัง (Behind the Scenes)
หากเราเปิด Terminal อีกหน้าต่าง แล้วลอง describe ดูที่ Pod server เราจะเห็น Event การสร้าง Ephemeral Container ปรากฏขึ้นมาอย่างชัดเจน:  

```bash
kubectl describe pod server 
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  2m33s  default-scheduler  Successfully assigned default/server
  Normal  Pulling    2m32s  kubelet            Pulling image "nginx"
  Normal  Pulled     2m32s  kubelet            Successfully pulled image "nginx" in 150ms 
  Normal  Created    2m32s  kubelet            Created container server
  Normal  Started    2m32s  kubelet            Started container server
  Normal  Pulling    2m26s  kubelet            Pulling image "busybox"
  Normal  Pulled     2m26s  kubelet            Successfully pulled image "busybox" in 163ms 
  Normal  Created    2m26s  kubelet            Created container debugger
  Normal  Started    2m26s  kubelet            Started container debugger
```

### บทสรุป
การใช้ Distroless Image คือ Best Practice ที่สมควรทำอย่างยิ่งสำหรับการขึ้น Production และด้วยความสามารถของ kubectl debug การแก้ไขปัญหาก็ไม่ได้เป็นเรื่องน่าปวดหัวอีกต่อไป
