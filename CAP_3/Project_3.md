#

## Array2D.h
```c++
#ifndef INCLUDED_ARRAY_2D_H
#define INCLUDED_ARRAY_2D_H

template<typename T>
class Array2D {
	// 二维数组
	// 宽 * 高
	//  x 0 - 10
	// y
	// 0
	// |
	// 10
private:
	T* mArray;
	int mWidth;
	int mHeight;
public:
	Array2D()
		: mArray(nullptr), mWidth(0), mHeight(0) {

	}
	~Array2D() {
		delete[] mArray;
		mArray = nullptr;
	}
	void SetSize(int width, int height) {
		if (mArray) {
			delete[] mArray;
			mArray = nullptr;
		}
		mWidth = width;
		mHeight = height;
		mArray = new T[width * height];
	}
	T& operator()(int curCol, int curRow) {
		return mArray[curRow * mWidth + curCol];
	}
	const T& operator()(int curCol, int curRow) const {
		return mArray[curRow * mWidth + curCol];
	}
};

#endif
```

## File.h
```c++
#pragma once

class File { // 读取DDS文件
private:
	int mSize;
	char* mData;
public:
	File(const char* filename);
	~File();
	int Size() const;
	const char* Data() const;
	unsigned int GetUnsigned(int pos) const;

};
```

## File.cpp
```c++
#include "File.h"
#include <fstream>

File::File(const char* filename) 
	: mSize(0), mData(nullptr) {
	std::ifstream in = std::ifstream(filename, std::ifstream::binary);
	if (in) {
		in.seekg(0, std::ifstream::end);
		mSize = static_cast<int>(in.tellg());
		in.seekg(0, std::ifstream::beg);
		mData = new char[mSize];
		in.read(mData, mSize);
	}
}
File::~File() {
	delete[] mData;
	mData = nullptr;
}
int File::Size() const {
	return mSize;
}
const char* File::Data() const {
	return mData;
}
unsigned int File::GetUnsigned(int pos) const {
	// 因为对齐的原因，需要读取 4个单个字节 然后合成为 4个字节
	// dds文件是小端存储，12 34 56 78 -> 78 56 34 12
	const unsigned char* up = reinterpret_cast<const unsigned char*>(mData);
	// 00 00 00 00
	// small -> max
	unsigned int res = 0;
	for (int i = 0; i < 4; i++) {
		unsigned int r = up[pos];
		r <<= i * 8;
		res |= r;
		pos += 1;
	}
	return res;
}
```

## Image.h
```c++
#pragma once

class Image { // 显示图片
private:
	int mWidth;
	int mHeight;
	unsigned int* mData;
public:
	Image(const char* filename);
	~Image();
	int Width() const; // 获取宽度
	int Height() const; // 获取高度
	void Draw(
		int dstX, int dstY,
		int srcX, int srcY,
		int width, int height) const;
};
```

## Image.cpp
```c++
#include "Image.h"
#include "File.h"
#include "GameLib/Framework.h"

Image::Image(const char* filename) 
	: mWidth(0), mHeight(0), mData(nullptr) {
	File f = File(filename);
	mHeight = f.GetUnsigned(12);
	mWidth = f.GetUnsigned(16);
	// 共有 mHeight * mWidth个像素，每个像素用一个 unsigned int 表示
	mData = new unsigned int[mHeight * mWidth];
	for (int i = 0; i < mHeight * mWidth; i++) {
		mData[i] = f.GetUnsigned(128 + 4 * i); // 128是因为偏置，4是因为unsigned int 为 4 byte
	}
}
Image::~Image() {
	delete[] mData;
	mData = nullptr;
}
int Image::Width() const {
	return mWidth;
}
int Image::Height() const {
	return mHeight;
}
void Image::Draw(
	int windowCol, int windowRow,
	int imageCol, int imageRow,
	int imageWidth, int imageHeight) const {
	unsigned int *vram = GameLib::Framework::instance().videoMemory();
	unsigned int windowWidth = GameLib::Framework::instance().width();
	for (int row = 0; row < imageHeight; row++) {
		for (int col = 0; col < imageWidth; col++) {
			int windowPos = (row + windowRow) * windowWidth + (col + windowCol);
			int imagePos = (row + imageRow) * mWidth + (col + imageCol);

      // 直接渲染
      //vram[windowPos] = mData[imagePos];

      // 如果前景 alpha >= 128, 才渲染
			/*if (mData[imagePos] & 0x80000000) { // 根据alpha(最高位的1字节)选择是否渲染
				vram[windowPos] = mData[imagePos];
			}*/


      // 混合
			unsigned int src = mData[imagePos];
			unsigned int srcA = (src & 0xff000000U) >> 24;
			unsigned int srcR = src & 0x00ff0000U;
			unsigned int srcG = src & 0x0000ff00U;
			unsigned int srcB = src & 0x000000ffU;

			unsigned int dstR = vram[windowPos] & 0x00ff0000U;
			unsigned int dstG = vram[windowPos] & 0x0000ff00U;
			unsigned int dstB = vram[windowPos] & 0x000000ffU;

			unsigned int r = srcR * srcA / 255 + (255 - srcA) * dstR / 255;
			unsigned int g = srcG * srcA / 255 + (255 - srcA) * dstG / 255;
			unsigned int b = srcB * srcA / 255 + (255 - srcA) * dstB / 255;

			vram[windowPos] = (r & 0x00ff0000U) | (g & 0x0000ff00U) | (b & 0x000000ffU);
		}
	}
}
```

## State.h
```c++
#ifndef INCLUDED_STATE_H
#define INCLUDED_STATE_H

#include "Array2D.h"
//#include <unordered_map>
class Image;

class State {
private:
	enum ObjectType : int { // 地图元素，goal会有额外的数组记录，此处不需要类别
		OBJ_SPACE = 0, OBJ_WALL = 1, OBJ_BLOCK = 2,
		OBJ_MAN = 3, OBJ_UNKNOWN = 4
	};
	enum ImageType : int { // 图片元素, 读取图片时直接使用该值，所以枚举值不能变
		IMAGE_MAN = 0, IMAGE_WALL = 1, IMAGE_BLOCK = 2, IMAGE_BLOCK_ON_GOAL = 3,
		IMAGE_GOAL = 4, IMAGE_SPACE = 5,
	};
	//std::unordered_map<ImageType, int> imageT2pos;
	int mWidth;
	int mHeight;
	Array2D<ObjectType> mObjects;
	Array2D<bool> mGoalFlags;
	Image* mImage;

	void SetSize(const char* stageData, int size);
	void DrawCell(int col, int row, ImageType imageType) const;

public:
	State(const char* stageData, int size);
	~State();
	void Update(char input);
	void StateDraw() const;
	bool isWin() const;
};
#endif
```

## State.cpp
```c++
#include "State.h"
#include "Array2D.h"
#include "Image.h"

State::State(const char* stageData, int size)
	: mWidth(0), mHeight(0), mObjects(Array2D<ObjectType>()), mGoalFlags(Array2D<bool>()) {
	SetSize(stageData, size); // 根据读入的地图（字符串表示）获得宽和高

	mObjects.SetSize(mWidth, mHeight); // 根据地图的宽和高分配二维数组
	mGoalFlags.SetSize(mWidth, mHeight); // 根据地图的宽和高分配二维数组

	/*imageT2pos = {
		{IMAGE_MAN, 0}, {IMAGE_WALL, 1},
		{IMAGE_BLOCK, 2}, {IMAGE_BLOCK_ON_GOAL, 3},
		{IMAGE_GOAL, 4}, {IMAGE_SPACE, 5}
	};*/

	int curCol = 0, curRow = 0;
	for (int i = 0; i < size; i++) {
		ObjectType objT = OBJ_WALL; // 变量都应该初始化
		bool goalFlag = false; // 变量都应该初始化
		switch (stageData[i]) {
		case ' ': objT = OBJ_SPACE; break;
		case '.': objT = OBJ_SPACE; goalFlag = true; break;
		case '#': objT = OBJ_WALL; break;
		case 'o': objT = OBJ_BLOCK; break;
		case 'O': objT = OBJ_BLOCK; goalFlag = true; break;
		case 'p': objT = OBJ_MAN; break;
		case 'P': objT = OBJ_MAN; goalFlag = true; break;
		case '\n': objT = OBJ_UNKNOWN; curCol = 0; curRow += 1; break;
		default:
			objT = OBJ_UNKNOWN; break;
		}
		if (objT != OBJ_UNKNOWN) { // 合法OBJECT
			mObjects(curCol, curRow) = objT;
			mGoalFlags(curCol, curRow) = goalFlag;
			curCol += 1;
		}
	}
	//mImage = new Image("nimotsuKunImage.dds"); // 载入图片
	mImage = new Image("nimotsuKunImage2.dds"); // 载入图片
}
State::~State() {
	delete mImage;
	mImage = nullptr;
}
void State::Update(char input) {
	// get input's pos change
	int dRow = 0, dCol = 0;
	switch (input) {
		case 'w': dRow -= 1; break;
		case 's': dRow += 1; break;
		case 'a': dCol -= 1; break;
		case 'd': dCol += 1; break;
		default: break;
	}

	// find man in map's pos
	int manPosRow = -1, manPosCol = -1;
	for (int row = 0; row < mHeight; row++) {
		for (int col = 0; col < mWidth; col++) {
			if (mObjects(col, row) == OBJ_MAN) {
				manPosRow = row;
				manPosCol = col;
				break;
			}
		}
		if (manPosRow != -1) break;
	}

	int manNextPosRow = manPosRow + dRow;
	int manNextPosCol = manPosCol + dCol;
	// 检查是否走到地图边界
	if (manNextPosRow < 0 || manNextPosRow >= mHeight || manNextPosCol < 0 || manNextPosCol >= mWidth)
		return;

	if (mObjects(manNextPosCol, manNextPosRow) == OBJ_SPACE) {
		// next pos is space or goal
		mObjects(manNextPosCol, manNextPosRow) = OBJ_MAN;
		mObjects(manPosCol, manPosRow) = OBJ_SPACE;
	} else {
		// next pos is block
		int boxNextPosRow = manNextPosRow + dRow;
		int boxNextPosCol = manNextPosCol + dCol;
		// 检查是否推到地图边界
		if (boxNextPosRow < 0 || boxNextPosRow >= mHeight || boxNextPosCol < 0 || boxNextPosCol >= mWidth)
			return;
		if (mObjects(boxNextPosCol, boxNextPosRow) == OBJ_SPACE) {
			mObjects(boxNextPosCol, boxNextPosRow) = OBJ_BLOCK;
			mObjects(manNextPosCol, manNextPosRow) = OBJ_MAN;
			mObjects(manPosCol, manPosRow) = OBJ_SPACE;
		}
	}
}

/*void State::StateDraw_ed1() const {
	// 遍历 mObjects，mGoalFlags
	for (int row = 0; row < mHeight; row++) {
		for (int col = 0; col < mWidth; col++) {
			ObjectType obj = mObjects(col, row); // 获取 ObjectType
			bool goalFlag = mGoalFlags(col, row); // 获取 goalFlag
			ImageType imageT = IMAGE_SPACE; // 使用默认值初始化
			switch (obj) {
				case OBJ_SPACE:
					imageT = goalFlag ? IMAGE_GOAL : IMAGE_SPACE;
					break;
				case OBJ_WALL:
					imageT = IMAGE_WALL;
					break;
				case OBJ_BLOCK:
					imageT = goalFlag ? IMAGE_BLOCK_ON_GOAL : IMAGE_BLOCK;
					break;
				case OBJ_MAN:
					imageT = IMAGE_MAN;
					break;
			}
			DrawCell(col, row, imageT);
		}
	}
}*/

void State::StateDraw() const {
	// 遍历 mObjects，mGoalFlags
	for (int row = 0; row < mHeight; row++) {
		for (int col = 0; col < mWidth; col++) {
			ObjectType obj = mObjects(col, row); // 获取 ObjectType
			bool goalFlag = mGoalFlags(col, row); // 获取 goalFlag
			if (obj != OBJ_WALL) { // 如果不是墙，就绘制地面
				DrawCell(col, row, goalFlag ? IMAGE_GOAL : IMAGE_SPACE);
			}

			ImageType imageT = IMAGE_SPACE;
			switch (obj)
			{
			case OBJ_WALL:
				imageT = IMAGE_WALL;
				break;
			case OBJ_BLOCK:
				imageT = IMAGE_BLOCK;
				break;
			case OBJ_MAN:
				imageT = IMAGE_MAN;
				break;
			}
			if (imageT != IMAGE_SPACE)
				DrawCell(col, row, imageT);
		}
	}
}

void State::DrawCell(int col, int row, ImageType imageType) const {
	int imageCol = -1;
  // imageCol是在组合图中各个图片的具体位置
	switch (imageType) {
	case IMAGE_MAN: imageCol = 0; break;
	case IMAGE_WALL: imageCol = 1; break;
	case IMAGE_BLOCK: imageCol = 2; break;
	case IMAGE_GOAL: imageCol = 3; break;
	case IMAGE_SPACE: imageCol = 4; break;
	}
	int imageWidth = 32, imageHeight = 32;
	mImage->Draw(col * imageWidth, row * imageWidth, imageCol * imageWidth, 0, imageWidth, imageWidth);
}

bool State::isWin() const {
	for (int row = 0; row < mHeight; row++) {
		for (int col = 0; col < mWidth; col++) {
			if (mObjects(col, row) == OBJ_BLOCK && not mGoalFlags(col, row)) {
				return false;
			}
		}
	}
	return true;
}

void State::SetSize(const char* stageData, int size) {
	// 根据读入的地图（字符串表示）获得宽和高
	int row = 0;
	int col = 0;
	for (int i = 0; i < size; i++) {
		if (stageData[i] == ' ' || stageData[i] == '#' 
			|| stageData[i] == '.' || stageData[i] == 'o'
			|| stageData[i] == 'O' || stageData[i] == 'p'
			|| stageData[i] == 'P') {
			col += 1;
		} else if (stageData[i] == '\n') { // stageData[i] == '\n'
			row += 1;
			mWidth = mWidth > col ? mWidth : col;
			mHeight = mHeight > row ? mHeight : row;
			col = 0;
		}
	}
}
```

## Main.cpp
```c++
#include <iostream>
#include "GameLib/Framework.h"

#include "State.h"
#include "File.h"

//函数原型
void mainLoop();

//全局变量
State* gState = 0;

//用户封装函数。内容被抛出给mainLoop（）
namespace GameLib {
	void Framework::update() {
		mainLoop();
	}
}

void mainLoop() {
	//×按钮被按下了吗？
	if (GameLib::Framework::instance().isEndRequested()) {
		if (gState) {
			delete gState;
			gState = 0;
		}
		return;
	}

	//初始化第一帧。绘制第一个状态并完成。
	if (!gState) {
		File file("stageData.txt");
		if (!(file.Data())) { //没有数据！
			GameLib::cout << "stage file could not be read." << GameLib::endl;
			return;
		}
		gState = new State(file.Data(), file.Size());
		
		//第一绘制
		gState->StateDraw();
		return; //结束
	}
	
	//主循环
	//获取输入
	GameLib::cout << "a:left s:right w:up z:down. command?" << GameLib::endl; //操作说明
	char input;
	GameLib::cin >> input;
	//结束判断
	if (input == 'q') {
		delete gState;
		gState = 0;
		GameLib::Framework::instance().requestEnd();
		return;
	}
	//更新
	gState->Update(input);
	//绘制
	gState->StateDraw();

	if (gState->isWin()) {
		//庆祝消息
		GameLib::cout << "Congratulation! you win." << GameLib::endl;
		delete gState;
		gState = 0;
	}
}
```
