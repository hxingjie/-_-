#

```c++
#include <algorithm>
#include <fstream>
#include "GameLib/Framework.h"

//函数原型
void readFile(char** buffer, int* size, const char* filename);
void mainLoop();


//二维数组类
template<typename T>
class Array2D{
public:
	Array2D()
		: mArray(nullptr){

	}
	~Array2D(){
		delete[] mArray;
		mArray = nullptr;  //为安全起见，把指针值设为0
	}
	void setSize(int width, int height){
		mWidth = width;
		mHeight = height;
		mArray = new T[width * height];
	}
	T& operator()(int col, int row){
		return mArray[row * mWidth + col];
	}
	const T& operator()(int col, int row) const {
		return mArray[row * mWidth + col];
	}
private:
	T* mArray;
	int mWidth;
	int mHeight;
};

//状态类
class State{
private:
	enum Object {
		OBJ_SPACE,
		OBJ_WALL,
		OBJ_BLOCK,
		OBJ_MAN,

		OBJ_UNKNOWN,
	};
	void setSize(const char* stageData, int size);

	int mWidth;
	int mHeight;
	Array2D<Object> mObjects;
	Array2D<bool> mGoalFlags;
public:
	State(const char* stageData, int size);
	void update(char input);
	void draw() const;
	bool isWin() const;
};

//全局变量
State* gState = 0;

//用户封装的函数。具体处理放在mainLoop()中
namespace GameLib{
	void Framework::update(){
		mainLoop();
	}
}

void mainLoop(){
	//第一帧负责初始化。绘制好最初的状态后结束
	if (not gState){ 
		const char* filename = "stageData.txt";
		char* stageData;
		int fileSize;
		readFile(&stageData, &fileSize, filename);
		if (not stageData){
			GameLib::cout << "stage file could not be read." << GameLib::endl;
			return;
		}
		gState = new State(stageData, fileSize);
		//析构
		delete[] stageData;
		stageData = nullptr;
		//第一次绘制
		gState->draw();
		return; //直接结束本次处理
	}

	//主循环
	GameLib::cout << "a:left s:right w:up z:down. command?" << GameLib::endl; //提示游戏如何操作
	
	//获取输入值
	char input;
	GameLib::cin >> input;

	//刷新
	gState->update(input);

	//绘制
	gState->draw();

	if (gState->isWin()){
		//通关祝贺信息
		GameLib::cout << "Congratulation! you win." << GameLib::endl;
		delete gState;
		gState = nullptr;
	}
}

//---------------------下面是函数定义--------------

void readFile(char** buffer, int* size, const char* filename){
	std::ifstream in(filename, std::ifstream::binary);
	if (not in){
		*buffer = nullptr;
		*size = 0;
	}else{
		in.seekg(0, std::ifstream::end);
		*size = static_cast<int>(in.tellg());
		in.seekg(0, std::ifstream::beg);
		*buffer = new char[*size];
		in.read(*buffer, *size);
	}
}

State::State(const char* stageData, int size)
	: mWidth(0), mHeight(0) {	
	//检测容量
	setSize(stageData, size);
	//确保容量
	mObjects.setSize(mWidth, mHeight);
	mGoalFlags.setSize(mWidth, mHeight);
	int col = 0;
	int row = 0;
	for ( int i = 0; i < size; ++i ){
		Object t;
		bool goalFlag = false;
		switch (stageData[i]){
			case '#': t = OBJ_WALL; break;
			case ' ': t = OBJ_SPACE; break;
			case 'o': t = OBJ_BLOCK; break;
			case 'O': t = OBJ_BLOCK; goalFlag = true; break;
			case '.': t = OBJ_SPACE; goalFlag = true; break;
			case 'p': t = OBJ_MAN; break;
			case 'P': t = OBJ_MAN; goalFlag = true; break;
			case '\n': col = 0; row += 1; t = OBJ_UNKNOWN; break; //换行处理
			default: t = OBJ_UNKNOWN; break;
		}
		if (t != OBJ_UNKNOWN){ //这个if处理的意义在如果遇到未定义的元素值就跳过它
			mObjects(col, row) = t; //写入
			mGoalFlags(col, row) = goalFlag; //终点信息
			col += 1;
		}
	}
}

void State::setSize(const char* stageData, int size){
	//当前位置
	int col = 0, row = 0;
	for (int i = 0; i < size; ++i){
		if (stageData[i] == '#' || stageData[i] == ' ' || stageData[i] == 'o'
			|| stageData[i] == 'O' || stageData[i] == '.' || stageData[i] == 'p' || stageData[i] == 'P') {
			col += 1;
		} else if (stageData[i] == '\n') {
			row += 1;
			mWidth = std::max(mWidth, col);
			mHeight = std::max(mHeight, row);
			col = 0;
		}
	}
}

void State::draw() const {
	for (int row = 0; row < mHeight; row++){
		for (int col = 0; col < mWidth; col++){
			Object obj = mObjects(col, row);
			bool goalFlag = mGoalFlags(col, row);
			switch (obj) {
				case OBJ_SPACE: goalFlag ? GameLib::cout << '.' : GameLib::cout << ' '; break;
				case OBJ_WALL: GameLib::cout << '#'; break;
				case OBJ_BLOCK: goalFlag ? GameLib::cout << 'O' : GameLib::cout << 'o'; break;
				case OBJ_MAN: goalFlag ? GameLib::cout << 'P' : GameLib::cout << 'p'; break;
			}
		}
		GameLib::cout << GameLib::endl;
	}
}

void State::update(char input){
	//移动量变换
	int dCol = 0, dRow = 0;
	switch (input){
		case 'a': dCol = -1; break; //左
		case 'd': dCol = 1; break; //右
		case 'w': dRow = -1; break; //上
		case 's': dRow = 1; break; //下
	}

	//查询小人的坐标
	int curCol = -1, curRow = -1;
	for (int row = 0; row < mHeight; row++) {
		for (int col = 0; col < mWidth; col++) {
			if (mObjects(col, row) == OBJ_MAN) {
				curCol = col;
				curRow = row;
				break;
			}
		}
		if (curCol != -1) break;
	}
	//移动
	//移动后的坐标
	int nextPosCol = curCol + dCol;
	int nextPosRow = curRow + dRow;
	//判断坐标的极端值。不允许超出合理值范围
	if (nextPosCol < 0 || nextPosRow < 0 || nextPosCol >= mWidth || nextPosRow >= mHeight)
		return;
	
	if (mObjects(nextPosCol, nextPosRow) == OBJ_SPACE){ //A.该方向上是空白或者终点。则小人移动
		mObjects(nextPosCol, nextPosRow) = OBJ_MAN;
		mObjects(curCol, curRow) = OBJ_SPACE;
	}else if (mObjects(nextPosCol, nextPosRow) == OBJ_BLOCK ){ //B.如果该方向上是箱子。并且该方向的下下个格子是空白或者终点，则允许移动
		//检测同方向上的下下个格子是否位于合理值范围
		int boxNextPosCol = nextPosCol + dCol;
		int boxNextPosRow = nextPosRow + dRow;
		if (boxNextPosCol < 0 || boxNextPosRow < 0 || boxNextPosCol >= mWidth || boxNextPosRow >= mHeight) //按键无效
			return;
		
		if (mObjects(boxNextPosCol, boxNextPosRow) == OBJ_SPACE){
			//按顺序替换
			mObjects(boxNextPosCol, boxNextPosRow) = OBJ_BLOCK;
			mObjects(nextPosCol, nextPosRow) = OBJ_MAN;
			mObjects(curCol, curRow) = OBJ_SPACE;
		}
	}
}

//只要还存在一个goalFlag值为false就不能判定为通关
bool State::isWin() const {
	for (int row = 0; row < mHeight; row++){
		for (int col = 0; col < mWidth; col++){
			if (mObjects(col, row) == OBJ_BLOCK && not mGoalFlags(col, row)){
				return false;
			}
		}
	}
	return true;
}
```

