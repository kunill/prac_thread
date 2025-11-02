# prac_thread
🧩 1. 기본 구조 개념 정리
구분	개념	설명
스레드 (std::thread)	멀티태스킹의 기본 단위	하나의 프로그램에서 여러 작업을 동시에 실행할 수 있게 함
뮤텍스 (std::mutex)	임계구역 보호	여러 스레드가 동시에 같은 데이터에 접근할 때 데이터 충돌 방지
조건변수 (std::condition_variable)	대기/신호 메커니즘	“큐가 비어 있으면 대기, push되면 깨움” 같은 패턴 구현
원자 변수 (std::atomic)	스레드 안전한 bool/int	stop 플래그나 카운터에 사용, Lock 없이도 안전한 접근
큐(deque/list/queue)	이벤트 전달용 컨테이너	생산자(메인)→소비자(워커) 이벤트 전달에 사용
unordered_map (HashMap)	센서별 상태 저장	빠른 검색/갱신, 키 기반 접근에 유용
enum class	상태 구분	코드 가독성 및 안전한 상수 사용
chrono	시간 측정/로그 타임스탬프	chrono::system_clock::now() 등으로 시간 관리
⚙️ 2. 반드시 익혀야 할 문법 / 문법 포인트별 예시
① std::thread — 스레드 생성과 종료
#include <thread>
#include <iostream>
using namespace std;

void worker(int id) {
    cout << "Thread " << id << " started!\n";
}

int main() {
    thread t(worker, 1);  // 스레드 생성
    t.join();             // 스레드 종료 대기
    cout << "Main done.\n";
}


✅ 실습 포인트

join() : 스레드 종료까지 기다림

detach() : 백그라운드로 실행 (보통 join을 권장)

② std::mutex — 공유 데이터 보호
#include <mutex>
#include <thread>
#include <iostream>
using namespace std;

int counter = 0;
mutex mtx;

void increase() {
    for (int i = 0; i < 1000; ++i) {
        lock_guard<mutex> lock(mtx);  // RAII 방식 자동 unlock
        counter++;
    }
}

int main() {
    thread t1(increase), t2(increase);
    t1.join(); t2.join();
    cout << "Counter: " << counter << endl;  // 항상 2000 보장
}


✅ 실습 포인트

lock_guard는 예외 발생해도 자동으로 unlock됨.

③ std::condition_variable — 생산자/소비자 큐
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
using namespace std;

queue<int> q;
mutex mtx;
condition_variable cv;
bool done = false;

void producer() {
    for (int i = 0; i < 5; ++i) {
        {
            lock_guard<mutex> lock(mtx);
            q.push(i);
        }
        cv.notify_one();
        this_thread::sleep_for(200ms);
    }
    done = true;
    cv.notify_all();
}

void consumer() {
    while (true) {
        unique_lock<mutex> lock(mtx);
        cv.wait(lock, [] { return !q.empty() || done; });
        if (!q.empty()) {
            cout << "consume " << q.front() << endl;
            q.pop();
        } else if (done) break;
    }
}

int main() {
    thread t1(producer), t2(consumer);
    t1.join(); t2.join();
}


✅ 실습 포인트

wait 조건 람다(lambda) 필수 (!q.empty() || done)

notify_one() / notify_all() 시점 주의

④ std::unordered_map — 해시맵으로 상태 관리
#include <unordered_map>
#include <iostream>
#include <string>
using namespace std;

enum class State { Normal, Warning };

int main() {
    unordered_map<string, State> sensors;
    sensors["A"] = State::Normal;
    sensors["B"] = State::Warning;

    for (auto& [id, st] : sensors)
        cout << id << " = " << (st == State::Normal ? "Normal" : "Warning") << endl;
}

⑤ enum class 와 switch
enum class AlarmState { Normal, Raised, Acked, Reraised };

void handle(AlarmState s) {
    switch (s) {
        case AlarmState::Normal: cout << "정상\n"; break;
        case AlarmState::Raised: cout << "발생\n"; break;
        default: cout << "기타 상태\n"; break;
    }
}

⑥ chrono — 시간/대기
#include <chrono>
#include <thread>
#include <iostream>
using namespace std;

int main() {
    auto start = chrono::system_clock::now();
    this_thread::sleep_for(1s);
    auto end = chrono::system_clock::now();
    chrono::duration<double> diff = end - start;
    cout << "Elapsed: " << diff.count() << "s\n";
}

🧠 3. 학습/실습 루트 추천
단계	주제	목표
1️⃣	std::thread + mutex	멀티스레드 기초 및 데이터 보호 이해
2️⃣	condition_variable	생산자/소비자 패턴 완벽히 익히기
3️⃣	unordered_map, enum class	상태 관리용 자료구조 구성
4️⃣	위 모든 것을 통합하여 미니 “알람 매니저” 실습	스레드 + 상태 관리 + 큐 실전 통합
5️⃣	응용 확장	- UI와 연동
- DDS/MQTT 토픽 수신으로 확장
- 로그 시스템 추가
