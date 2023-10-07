---
title: Led灯控制(Usart)
description: 使用cpp完成一个串口屏对led和led矩阵的控制
date: 2023-10-07 00:00:00+0000
categories:
    - Stm   
tags:
    - TJC
    - CPP
license: 
hidden: false
comments: false
---

## CPP中的重定向printf

cpp不允许重定义 => 不按照cpp规则编译即可

```cpp
extern "C"
{
  int fputc(int ch, FILE *f)
  {
    HAL_UART_Transmit(&huart3, (uint8_t *)&ch, 1, 0xFFFF);
    return ch;
  }
}
```

## LED控制亮度 => LED 类

```cpp
class Led
{
private:
    TIM_HandleTypeDef *__htim__;
    unsigned int __ch__;
    int __max__ = 255;
    int __diff__ = 10;
    int CrtTask = 0;
    int CrtBreathStatus = 0;

public:
    int crt;
    
    Led(TIM_HandleTypeDef *htim, unsigned int ch)
    {
        __htim__ = htim;
        __ch__ = ch;
    };

    ~Led()
    {
        __HAL_TIM_SET_COMPARE(__htim__, __ch__, 0);
    };

    void set(int compare)
    {
        if (compare < 0)
        {
            compare = 0;
        }
        else if (compare > __max__)
        {
            compare = __max__;
        }
        crt = compare;
        __HAL_TIM_SetCompare(__htim__, __ch__, compare);
    }

    void reset()
    {
        set(0);
        CrtTask = 0;
        CrtBreathStatus = 0;
    }

    void toggle(int compare)
    {
        if (crt == 0)
        {
            set(compare);
        }
        else
        {
            reset();
        }
    }

    bool minus(int diff)
    {
        if (crt == 0)
        {
            return false;
        }
        else
        {
            set(crt - diff);
            return true;
        }
    }

    bool add(int diff)
    {
        if (crt == __max__)
        {
            return false;
        }
        else
        {
            set(crt + diff);
            return true;
        }
    }

    bool TryBreath(int diff, bool always = true)
    {
        switch (CrtBreathStatus)
        {
        case 1:
            if (!add(diff))
            {
                CrtBreathStatus = 2;
            }
            return true;
        case 2:
            if (!minus(diff))
            {
                if (always)
                {
                    CrtBreathStatus = 1;
                }
                else
                {
                    reset();
                }
            }
            return true;
        default:
            return false;
        }
    }

    void setTask(int task)
    {
        CrtTask = task;
    }

    void execTask()
    {
        switch (CrtTask)
        {
        case 1:
            add(__diff__);
            break;
        case 2:
            minus(__diff__);
            break;
        case 3:
            if (CrtBreathStatus == 0)
                CrtBreathStatus = 1;
            TryBreath(__diff__);
            break;
        case 4:
            if (CrtBreathStatus == 0)
                CrtBreathStatus = 1;
            TryBreath(__diff__, false);
            break;
        case 0:
            break;
        default:
            break;
        }
    }
};
```

## LED类 => LED Array 类

简单的将上面的LED用array来 遍历 控制。

```cpp
template <size_t N>
class LedArray
{
public:
    std::array<Led, N> __leds__;
    unsigned int __size__;
    
    LedArray(std::array<Led, N> &leds) : __leds__(leds), __size__(N){};
    
    ~LedArray()
    {
        for (Led &led : __leds__)
        {
            led.reset();
        }
    };

    void resetAll()
    {
        for (Led &led : __leds__)
        {
            led.reset();
        }
    }
    void printCompares()
    {
        int i = 0;
        for (Led &led : __leds__)
        {
            printf("index: %d, compare: %d\n", i, led.crt);
            i += 1;
        }
    }
    void setCompares(int compares[])
    {
        customAssert(getArrayLength(compares), __size__, Comparison::Equal);
        int i = 0;
        for (Led &led : __leds__)
        {
            led.set(compares[i]);
            i += 1;
        }
    }
    void toggleLeds(int compares[])
    {
        customAssert(getArrayLength(compares), __size__, Comparison::Equal);
        int i = 0;
        for (Led &led : __leds__)
        {
            if (compares[i])
            {
                led.toggle(compares[i]);
            }
            i += 1;
        }
    }
    void setTasks(int tasks[])
    {
        customAssert(getArrayLength(tasks), __size__, Comparison::Equal);
        int i = 0;
        for (Led &led : __leds__)
        {
            led.setTask(tasks[i]);
            i += 1;
        }
    }
    void execTasks()
    {
        for (Led &led : __leds__)
        {
            led.execTask();
        }
    }
};
```

## 延时执行类 Delay
为了实现 流水 效果， 需要延时(每隔几次运行一次)来执行

hpp
```cpp
class Delay
{
public:
    Delay(int delayTimes = 5, bool first = true);
    ~Delay();
    void reset();
    void ctn();
    bool exceed();

private:
    int __delay_times__;
    int __crt_times__;
};
```
cpp
```cpp
Delay::Delay(int delayTimes, bool first) : __delay_times__(delayTimes), __crt_times__(delayTimes) {}

Delay::~Delay()
{
    __delay_times__ = 0;
}

void Delay::reset()
{
    __crt_times__ = 0;
}

void Delay::ctn()
{
    __crt_times__ += 1;
}

bool Delay::exceed()
{
    if (__crt_times__ >= __delay_times__)
    {
        reset();
        return true;
    }
    else
    {
        ctn();
        return false;
    }
}
```

##　串口屏控制(状态控制和直接控制)
这里的状态一是Crt_Main_Task, 也就写了个流水，二是每个灯独立的状态控制，用到的也就一个 呼吸。

```cpp
void reset_whole()
{
  Crt_Main_Task = 0;
  CrtLedIndex = 0;
  ledarray.resetAll();
  delay_20.reset();
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
  if (huart == &huart1)
  {
    __HAL_UART_CLEAR_IT(&huart1, UART_CLEAR_IDLEF);
    __HAL_UART_DISABLE_IT(&huart1, UART_IT_TXE);
    Crt_Main_Task = 0;
    
    switch (rev_data[0])
    {
    // 全关复位
    case 0x01:
      reset_whole();
      break;
    // 设置亮度
    case 0x02:
      int compares[8];
      for (int i = 0; i < 8; i++)
      {
        compares[i] = rev_data[i + 1];
      }
      ledarray.setCompares(compares);
      break;
    // 亮灭互转且设置亮度
    case 0x03:
      int toggles[8];
      for (int i = 0; i < 8; i++)
      {
        toggles[i] = rev_data[i + 1];
      }
      ledarray.toggleLeds(toggles);
      break;
    // 呼吸(逐渐亮/暗)
    case 0x04:
      int tasks[8];
      for (int i = 0; i < 8; i++)
      {
        tasks[i] = 3 * rev_data[i + 1];
      }
      ledarray.setTasks(tasks);
      break;
    // 流水呼吸
    case 0x05:
      Crt_Main_Task = 1;
      // 这个函数用来处理各led的顺序大小然后重新排序
      sortVectorByValue(rev_data, 8, Blink_Leds, Blink_Cnt); 
    }

    __HAL_UART_ENABLE_IT(&huart1, UART_IT_TXE);
    HAL_UART_Receive_IT(&huart1, rev_data, 9);
    __HAL_UART_FLUSH_DRREGISTER(&huart1);
    __HAL_UART_CLEAR_FLAG(&huart1, UART_FLAG_RXNE);
  }
}
```

上面的sortVectorByValue，这直接 Ai 写吧，先排个序然后再判断相邻的是不是相等就行了。

```cpp
void sortVectorByValue(const uint8_t vec[], int size, int sortedIndices[][MAX_SIZE],
                       int &count)
{
    count = 0;
    for (int i = 0; i < MAX_SIZE; i++)
    {
        for (int j = 0; j < MAX_SIZE; j++)
        {
            sortedIndices[i][j] = -1;
        }
    }
    Pair indexedVec[MAX_SIZE];
    for (int i = 0; i < size; i++)
    {
        indexedVec[i].index = i;
        indexedVec[i].value = (int)vec[i + 1];
    }

    std::sort(indexedVec, indexedVec + size, compareByValue);

    int subArray[MAX_SIZE];
    int subCount = 0;
    int prevValue = indexedVec[0].value;

    for (int i = 0; i < size; i++)
    {
        if (indexedVec[i].value == prevValue)
        {
            subArray[subCount++] = indexedVec[i].index;
        }
        else
        {
            for (int j = 0; j < subCount; j++)
            {
                sortedIndices[count][j] = subArray[j];
            }
            count++;
            subCount = 0;
            subArray[subCount++] = indexedVec[i].index;
            prevValue = indexedVec[i].value;
        }
    }

    for (int j = 0; j < subCount; j++)
    {
        sortedIndices[count][j] = subArray[j];
    }
    count++;
}
```
## 定时执行(执行状态对应的任务)

```cpp
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
  if (htim == &htim2)
  {
    // 执行Crt_Main_Task对应的任务，这个其实可以再拆一个 类 出来，但是没有更多功能要写,
    // 先扔这里了。
    switch (Crt_Main_Task)
    {
    case 1:
      // 延时开启下一组led的呼吸任务
      if (delay_20.exceed())
      {
        if (CrtLedIndex >= Blink_Cnt)
        {
          CrtLedIndex = 0;
        }
        else
        {
          for (int &index : Blink_Leds[CrtLedIndex])
          {
            if (index >= 0 & index < 8)
            {
              ledarray.__leds__[index].setTask(4);
            }
          }
          CrtLedIndex += 1;
        }
      }
      break;
    default:
      break;
    }
    // 执行所有的led的任务
    ledarray.execTasks();
    // 隔一段时间输出一下led的状态
    if (delay_30.exceed())
      ledarray.printCompares();
  }
}
```