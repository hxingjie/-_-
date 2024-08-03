#

## 位操作存储状态
```c++
#include <iostream>
#include <vector>

class Flag {
public:
    Flag()
        : mFlag(0), mGuard(1) {

    }
    bool check(unsigned char pos) const {
        return mFlag & (mGuard << pos);
    }
    bool check(std::vector<unsigned char> &pos) const {
        unsigned char tmp = 0;
        // 0000 0000 | 1000 0000 -> 1000 0000
        // 1000 0000 | 0100 0000 -> 1100 0000
        for (const unsigned char &t : pos) {
            tmp |= mGuard << t;
        }
        return mFlag & tmp;
    }
    void set(unsigned char pos) {
        mFlag |= (mGuard << pos);
    }
    void reset(unsigned char pos) {
        mFlag &= ~(mGuard << pos);
    }
public:
    unsigned char mFlag;
    unsigned char mGuard;
};

int main() {
    Flag flag;
    bool b = flag.check(6);
    flag.set(6);
    b = flag.check(6);
    flag.reset(6);
    b = flag.check(6);

    return 0;
}
```
