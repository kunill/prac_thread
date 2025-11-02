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

🧩 Step 1. Thread + Mutex 기초 실습
🎯 목표

스레드가 동시에 동작하는 것을 직접 눈으로 확인

std::mutex로 공유 데이터 보호하는 원리 이해

📘 코드 예제
#include <iostream>
#include <thread>
#include <mutex>
using namespace std;

int g_counter = 0;     // 모든 스레드가 공유하는 전역 변수
mutex g_mtx;           // 뮤텍스: 데이터 보호용

void worker(const string& name) {
    for (int i = 0; i < 5; ++i) {
        {
            lock_guard<mutex> lock(g_mtx); // 잠금 시작
            ++g_counter;
            cout << "[" << name << "] 작업 중... g_counter=" << g_counter << endl;
        } // lock 해제 (스코프 종료 시 자동 unlock)
        this_thread::sleep_for(200ms);
    }
}

int main() {
    cout << "=== Step1: Thread + Mutex Test ===" << endl;

    thread t1(worker, "스레드1");
    thread t2(worker, "스레드2");

    // 스레드가 끝날 때까지 대기
    t1.join();
    t2.join();

    cout << "모든 스레드 종료. 최종 counter=" << g_counter << endl;
    return 0;
}

🧠 학습 포인트
개념	설명
std::thread	새로운 실행 흐름 생성
join()	스레드 종료까지 기다림
mutex	동시 접근을 막기 위한 잠금 객체
lock_guard<mutex>	RAII 방식으로 잠금/해제 자동 관리
sleep_for()	스레드 실행 간 딜레이 (작업 시뮬레이션)
✅ 실습 방법

파일 이름: step1_thread_mutex.cpp

빌드:

g++ -std=c++17 -pthread step1_thread_mutex.cpp -o step1


실행:

./step1

🔍 실행 결과 예시
=== Step1: Thread + Mutex Test ===
[스레드1] 작업 중... g_counter=1
[스레드2] 작업 중... g_counter=2
[스레드1] 작업 중... g_counter=3
[스레드2] 작업 중... g_counter=4
...
모든 스레드 종료. 최종 counter=10


만약 mutex를 제거하면?
→ 스레드들이 동시에 g_counter를 바꾸다가 값이 꼬여서
최종 counter=10이 아니라 8~9 정도가 나올 수 있음 (race condition).


🧩 Step 2. ThreadSafeQueue로 생산자/소비자 패턴 익히기
🎯 목표

std::condition_variable로 “대기/깨우기” 패턴 이해

안전한 ThreadSafeQueue<T> 구현

생산자(Producer) 스레드 → 소비자(Consumer) 스레드 데이터 전달

📘 예제 코드 (단일 파일)

파일명: step2_threadsafequeue.cpp
빌드:

g++ -std=c++17 -O2 -pthread step2_threadsafequeue.cpp -o step2


실행:

./step2

#include <bits/stdc++.h>
using namespace std;

// ===================== ThreadSafeQueue ======================
// - push(): 생산자가 데이터 넣고 notify_one()으로 소비자 깨움
// - wait_and_pop(): 큐가 빌 때 condition_variable로 대기
// - stop 플래그로 안전 종료

template <typename T>
class ThreadSafeQueue {
public:
    void push(T v) {
        {
            lock_guard<mutex> lk(mx_);
            q_.push_back(std::move(v));
        }
        cv_.notify_one();
    }

    // 대기 후 pop (stop=true 이고 q 비었으면 false)
    bool wait_and_pop(T& out, atomic<bool>& stop) {
        unique_lock<mutex> lk(mx_);
        cv_.wait(lk, [&]{ return !q_.empty() || stop.load(); });
        if (stop.load() && q_.empty()) return false;
        out = std::move(q_.front());
        q_.pop_front();
        return true;
    }

    // 비차단 pop (필요시 사용)
    bool try_pop(T& out) {
        lock_guard<mutex> lk(mx_);
        if (q_.empty()) return false;
        out = std::move(q_.front());
        q_.pop_front();
        return true;
    }

    bool empty() const {
        lock_guard<mutex> lk(mx_);
        return q_.empty();
    }

private:
    mutable mutex mx_;
    condition_variable cv_;
    deque<T> q_;
};

// ===================== 예제: int 메시지 흐름 =====================

struct Msg {
    int producerId;
    int value;
    chrono::system_clock::time_point ts;
};

static string tstr(chrono::system_clock::time_point tp) {
    time_t t = chrono::system_clock::to_time_t(tp);
    tm bt{};
#if defined(_WIN32)
    localtime_s(&bt, &t);
#else
    localtime_r(&t, &bt);
#endif
    char buf[32];
    strftime(buf, sizeof(buf), "%H:%M:%S", &bt);
    return buf;
}

int main() {
    ThreadSafeQueue<Msg> q;
    atomic<bool> stop{false};

    // 소비자(워커) 스레드: 큐에서 메시지를 소비
    thread consumer([&]{
        while (!stop.load()) {
            Msg m{};
            if (!q.wait_and_pop(m, stop)) break; // stop+빈큐면 종료
            cout << "[CONSUMER " << tstr(chrono::system_clock::now()) << "] "
                 << "from P" << m.producerId << " value=" << m.value << "\n";
            // 처리 시뮬레이션
            this_thread::sleep_for(150ms);
        }
        cout << "[CONSUMER] exit.\n";
    });

    // 생산자 2개 예시: 각자 다른 주기로 메시지 생산
    auto producerFn = [&](int id, int count, chrono::milliseconds period) {
        for (int i = 1; i <= count; ++i) {
            q.push(Msg{ id, i, chrono::system_clock::now() });
            cout << "  [PRODUCER " << id << "] push " << i << "\n";
            this_thread::sleep_for(period);
        }
    };

    thread p1(producerFn, 1, 5, 200ms);
    thread p2(producerFn, 2, 5, 120ms);

    p1.join();
    p2.join();

    // 모든 생산자 종료 → stop=true로 바꿔서 소비자 정리
    stop.store(true);
    // 소비자가 wait 중일 수 있으므로 dummy notify (빈 push도 가능)
    q.push(Msg{ -1, -1, chrono::system_clock::now() });

    consumer.join();
    cout << "Main exit.\n";
    return 0;
}

🔍 핵심 포인트 해설

cv_.wait(lock, pred):
큐가 비어있을 때만 대기하고, push() 또는 stop이 켜지면 깨어납니다.
항상 조건 람다를 사용해 “spurious wakeup(가짜 깨움)”에 안전해야 합니다.

정리(Shutdown) 패턴:
생산자가 끝나면 stop=true로 바꾸고, 대기 중인 소비자를 깨우기 위해 notify가 필요합니다. 여기선 간단히 push()로 깨웁니다.

deque<T>:
앞뒤에서 빠르게 push/pop이 가능한 컨테이너. 일반 큐(std::queue)로 감싸도 무방하지만, 예제에선 직접 사용했습니다.

멀티 프로듀서/싱글 컨슈머(MPSC):
위 예제는 생산자 2개, 소비자 1개 구조. ThreadSafeQueue가 내부에서 잠금으로 보호되므로 안전합니다.

🧪 실습 체크리스트

주기 바꾸기: producerFn의 주기(200ms, 120ms)를 바꿔 메시지 interleave 확인

컨슈머 2개로 늘리기: 소비자 스레드를 2개 만들고 동일 큐에서 꺼내 처리 (경쟁적 소비)

배치 처리: 소비자가 큐에서 메시지를 모아서 5개 단위로 묶음 처리해보기

try_pop 사용: 일정 시간마다 try_pop으로 비동기 폴링(권장X)과 wait_and_pop의 차이 체험

종료 시그널 별도화: stop 대신 “특수 메시지(예: producerId=-1)”를 보내 종료하는 방법으로 바꿔보기

