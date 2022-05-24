### File structure
molecular/ 
- _init_.py
- descriptions.py            CONSTANT VARIABLE เกี่ยวกับคำอธิบาย
- names.py                   CONSTANT VARIABLE เกี่ยวกับชื่อ
- operators.py               ไฟล์ที่ทำการรัน simulate จำลองการทำ mocular โดยมีเรียกใช้ไฟล์ core.pyx แล้วนำค่าที่ได้กลับมาใส่ใน       blender particle system
-  simulate.py                ใส่ค่าเริ่มต้นให้กับระบบ simulate
-  ui.py                      หน้าต่าง UI ใน blender
-  utils.py                   ฟังก์ชันช่วยเพิ่มเติม

source/
-  core.pyx                  ไฟล์หลักในการคำนวณค่าต่างๆ ก่อนจะ return importdata ที่คำนวณแล้วกลับไป
-  setup.bat                 compiling core.pyx, ย้ายไฟล์และลบไฟล์ที่ไม่จำเป็น
-  setup.py                  ตั้งค่า complie สำหรับ cython

make_release.py               ใช้รันสำหรับสร้าง file zip addon และพร้อใสำหรับใช้ใน blender - คำสั่ง 'python make_release.py' 

---

### core.pyx function สำคัญ
- init(importdata) ตั้งค่าเริ่มต้นการคำนวณ, ถูกเรียกใข้งานเพียงครั้งเดียวเมื่อ addon เริ่มทำงาน (ถูกเรียกใช้จาก operators.py)
- simulate(importdata) ฟังก์ชันหลักจำลองการทำงานของ molecular, ถูกเรียกใช้งาน 1ครั้ง/framesubstep  (ฟังก์ชันชันที่ 2 ที่ถูกเรียกใช้โดย operators.py)
- update(data) นำค่าที่นำเข้ามาจาก importdata ของฟังก์ชัน simulate เข้าไปอัพเดรตข้อมูลปัจจุบันก่อนเริ่มการคำนวณ
- create_link(int par_id, int max_link, int parothers_id=-1) สร้าง link ระหว่าง particle
- solve_link(Particle *par) - คำนวณค่า particle ใน link
- collide(Particle *par) - คำนวณค่า self-collision ระหว่าง particle
- KDTree_rnn_search - neighbor search

---------
ฟังก์ชันที่ถูกสร้างเพิ่มเติมขึ้นมา
- remove_link(Particle *par) ลบ link ระหว่าง particle (ถูกเรียกใช้โดย create_link)

---
## core.pyx
cpdef init(importdata):
- Description: ตั้งค่าเริ่มต้นให้กับระบบ simulate
- Parameters: importdata (Array[]) – ข้อมูลที่นำเข้าจาก blender ไปยังระบบ simulate.
- Returns: parnum, number of particle.
- Return type: Int

cdef testkdtree(int verbose = 0):
- Description: ทดสอบ k-d tree โดยการ print output ออกมาตรวจสอบ
- Parameters: verbose (Int) – สถานะที่ใช้บอกในการแสดงรายละเอียด ถ้าค่า >= 1 บอกจำนวน particle ที่พบ, ถ้าค่า >= 2 ให้แสดง particle id ที่หาเจอ และถ้าค่า >= 3 แสดง (Parent, Particle id, Left, Right)

cpdef simulate(importdata):
- Description: จำลองระบบ molecular โดยรับค่า velocity และค่าอื่นๆที่จำเป็นมาก่อนจะคืนค่าที่ update แล้วกลับไป
- Parameters: importdata (Array[]) – ข้อมูล particles ที่นำเข้าจาก blender ไปยังระบบ simulate.
- Returns: exportdata - ข้อมูล particle ที่คำนวณใหม่เรียบร้อย
- Return type: Array[]

cpdef memfree():
- Description: ทำการ reset ค่า fps, substep, deltatime, และค่าอื่นๆในระบบ และทำใช้ฟังกชัน free กับทุก node ใน kd tree

cdef void collide(Particle *par)nogil:
- Description: คำนวณค่า velocity ของ particle หลังจากการชนกันระหว่าง particle
- Parameters: par (Particle) – ข้อมูลของ node particle

cdef void solve_link(Particle *par, int state)nogil:
- Description: คำนวณค่า velocity ของ particle หลังจากระหว่าง link
- Parameters: 
  - par (Particle) – ข้อมูลของ node particle
  - par (state) – ถ้ามีค่ามากกว่าเท่ากับ 1 จะทำการเรียกการทำงาน remove_link ด้วย

cdef void update(data):
- Description: นำค่า importdata ไป update ค่า velocity และข้อมูลอื่นๆ ของ particle ในระบบและ update ข้อมูลใน kdtree
- Parameters: data (Array[]) – ข้อมูล importdata ที่ถูกส่งต่อมาจากตัวแปร importdata ในฟังก์ชัน simulate โดยตรง

cdef void KDTree_create_nodes(KDTree *kdtree,int parnum):
- Description: สร้าง KDTree จำนวน node >= parnum
- Parameters: 
  - kdtree (KDTree) – pointer ของ kdtree ที่ต้องการให้สร้าง
  - parnum (Int) – จำนวน parnum ใข้ในการกำหนดจำนวน node ที่จะสร้าง

cdef Node KDTree_create_tree(
        KDTree *kdtree,
        SParticle *kdparlist,
        int start,
        int end,
        int name,
        int parent,
        int depth,
        int initiate
        )nogil:
- Description: สร้าง kdtree จาก kdparlist (sequence ที่เก็บข้อมูลของ particle ทั้งหมด)
- Parameters: 
  - kdtree (KDTree) – pointer ของ kdtree ที่ต้องการให้เริ่มสร้าง
  - kdparlist (SParticle) - ข้อมูล particle ที่ถูกสร้างขึ้นโดยการ copy มาจากข้อมูลหลักใช้ในการสร้าง kdtree
  - start (int) - index ที่ต้องการให้เริ่มใน kdparlist
  - end (int) - index สุดท้ายที่ต้องการใน kdparlist
  - name (int) - name ของ node
  - parent (int) - par_id ของ particle ที่เป็น parent
  - depth (int) - ความสูงของ kdtree
  - initiate (int) - ถ้ามีค่า 1 จะมีการใส่ข้อมูลลงใน node
- Returns: kdtree node - node ที่เติมข้อมูล
- Return type: Node
 
cdef int KDTree_rnn_query(
        KDTree *kdtree,
        Particle *par,
        float point[3],
        float dist
        )nogil:
- Description: ค้นหา particle neighbours รอบ par node
- Parameters: 
  - kdtree (KDTree) - kdtree ที่เก็บข้อมูล particle ใช้สำหรับค้นหาข้อมูล particle ต่างๆ
  - par (Particle) - ข้อมูล Particle หลักที่จะค้นหาว่ามี particle neighbours อะไรบ้าง
  - point (float[3]) - ตำแหน่งของ particle ที่จะค้นหา particle neighbours
  - dist (float) - ความยาวในการค้นหา neighbours search

cdef void KDTree_rnn_search(
        KDTree *kdtree,
        Particle *par,
        Node node,
        float point[3],
        float dist,
        float sqdist,
        int k,
        int depth
        )nogil:
- Description: ค้นหา nearest neighbour ของ particle 
- Parameters: 
  - kdtree (KDTree) – ข้อมูล particle ทั้งหมดสำหรับค้นหา
  - par (Particle) – ข้อมูลของ Particle
  - node (Node) - ข้อมูล node ของ particle ที่กำลังค้นหา
  - point (float[3]) - ตำแหน่งของ particle ที่จะค้นหา particle neighbours
  - dist (float) - ความยาวในการค้นหา neighbours search
  - sqdist (float) - dist*dist
  - k (int) - ไม่มีการใช้ค่านี้
  - depth (int) - ความลึกของ kdtree 

@vesion(1.1.3)
cdef void remove_link(Particle *par)nogil:
- Description: ลบคู่ link ที่อยู่ระหว่าง par และ node อื่นๆ ภายโต้เงื่อนไขว่าต้องเป็นขอบของวัตถุ
- Parameters: par (Particle) – ข้อมูลของ Particle 

@vesion(1.1.3)
cdef void remove_link_all()nogil:
- Description: ทำหน้าที่ลบ link ระหว่าง particle ทั้งหมดโดยเรียกใช้ remove_link(Particle *par)

cdef void create_link(int par_id, int max_link, int parothers_id=-1)nogil:
- Description: สร้าง link ระหว่าง par_id กับ parother_id หรือ neighbours ของ par_id
- Parameters: 
  - par_id (int) – id หรือ index ไว้ในการเข้าถึง Particle ใน global array variable 
  - max_link (int) - จำนวน link มากที่สุดที่สามารถมีได้ต่อหนึ่ง par_id
  - parothers_id (int) - par_id ของ Particle อื่นที่ต้องการให้ link กับอันหลัก
  
---
## simulate.py
def pack_data(context, initiate):
- Description: นำค่าจาก setting ใส่ Array[] ให้พร้อมสำหรับการนำเข้าระบบ simulate
- Parameters: 
  - initiate (boolean) - ถ้าเป็น True จะมีการใส่ค่าข้อมูล params ลงไปใน exportdata
  - context (bpy.context)
- Returns: mol_exportdata - array ข้อมูลสำหรับจำลอง molecular
- Return type: Array[]

---
## util.py
def get_object(context, obj):
- Description: 
- Parameters: 
- Returns: 
- Return type: 

def destroy_caches(obj):
- Description: 
- Parameters: 
- Returns: 
- Return type: 

---
## operators.py
