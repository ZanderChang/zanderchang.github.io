---
title: LibFuzzer中的变异策略分析
mathjax: false
date: 2019-12-11 14:36:01
categories:
- 模糊测试
tags:
- LLVM
---

`LibFuzzer`是一个模糊测试引擎，是`LLVM`项目的一部分，在当前`LLVM`中默认被支持。目前`llvm-project`提供较为方便的编译方法。

<!-- more -->

## 简介

`LibFuzzer`是`LLVM`项目中`compiler-rt`的一部分，在[这里](https://llvm.org/docs/LibFuzzer.html)有其详细使用说明。其它使用说明包括Google的2篇文章[libFuzzerTutorial](https://github.com/google/fuzzing/blob/master/tutorial/libFuzzerTutorial.md)和[structure-aware-fuzzing](https://github.com/google/fuzzing/blob/master/docs/structure-aware-fuzzing.md)，中文翻译分别对应[这里](https://www.4hou.com/technology/16554.html)和[这里](https://www.4hou.com/technology/16633.html)。本文将分析`LibFuzzer`的变异策略，其关键代码位于[这里](https://github.com/llvm/llvm-project/blob/master/compiler-rt/lib/fuzzer/FuzzerMutate.cpp)。

目前`LibFuzzer`位于`LLVM`项目中，因此其编译方法只需在创建`Makefile`的过程中指定`compiler-rt`即可：`cmake -G "Unix Makefiles" -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;compiler-rt"`。

在代码开始处可以看到`Dispatcher`中其用到的所有变异策略，共有15种，下面依次进行分析。

```cpp
MutationDispatcher::MutationDispatcher(Random &Rand,
                                       const FuzzingOptions &Options)
    : Rand(Rand), Options(Options) {
  DefaultMutators.insert(
      DefaultMutators.begin(),
      {
          {&MutationDispatcher::Mutate_EraseBytes, "EraseBytes"},
          {&MutationDispatcher::Mutate_InsertByte, "InsertByte"},
          {&MutationDispatcher::Mutate_InsertRepeatedBytes,
           "InsertRepeatedBytes"},
          {&MutationDispatcher::Mutate_ChangeByte, "ChangeByte"},
          {&MutationDispatcher::Mutate_ChangeBit, "ChangeBit"},
          {&MutationDispatcher::Mutate_ShuffleBytes, "ShuffleBytes"},
          {&MutationDispatcher::Mutate_ChangeASCIIInteger, "ChangeASCIIInt"},
          {&MutationDispatcher::Mutate_ChangeBinaryInteger, "ChangeBinInt"},
          {&MutationDispatcher::Mutate_CopyPart, "CopyPart"},
          {&MutationDispatcher::Mutate_CrossOver, "CrossOver"},
          {&MutationDispatcher::Mutate_AddWordFromManualDictionary,
           "ManualDict"},
          {&MutationDispatcher::Mutate_AddWordFromPersistentAutoDictionary,
           "PersAutoDict"},
      });
  if(Options.UseCmp)
    DefaultMutators.push_back(
        {&MutationDispatcher::Mutate_AddWordFromTORC, "CMP"});

  if (EF->LLVMFuzzerCustomMutator)
    Mutators.push_back({&MutationDispatcher::Mutate_Custom, "Custom"});
  else
    Mutators = DefaultMutators;

  if (EF->LLVMFuzzerCustomCrossOver)
    Mutators.push_back(
        {&MutationDispatcher::Mutate_CustomCrossOver, "CustomCrossOver"});
}
```

## EraseBytes

随机删除`Data`中的`N`个数据。

```cpp
size_t MutationDispatcher::Mutate_EraseBytes(uint8_t *Data, size_t Size,
                                             size_t MaxSize) {
  if (Size <= 1) return 0;
  size_t N = Rand(Size / 2) + 1;
  assert(N < Size);
  size_t Idx = Rand(Size - N + 1);
  // Erase Data[Idx:Idx+N].
  memmove(Data + Idx, Data + Idx + N, Size - Idx - N);
  // Printf("Erase: %zd %zd => %zd; Idx %zd\n", N, Size, Size - N, Idx);
  return Size - N;
}
```

## InsertByte

在`Data`的一个随机位置插入一个随机字符。

```cpp
size_t MutationDispatcher::Mutate_InsertByte(uint8_t *Data, size_t Size,
                                             size_t MaxSize) {
  if (Size >= MaxSize) return 0;
  size_t Idx = Rand(Size + 1);
  // Insert new value at Data[Idx].
  memmove(Data + Idx + 1, Data + Idx, Size - Idx);
  Data[Idx] = RandCh(Rand);
  return Size + 1;
}
```

## InsertRepeatedBytes

在`Data`中的一个随机位置处连续插入`N`（至少`3`个）个相同的随机字符。

```cpp
size_t MutationDispatcher::Mutate_InsertRepeatedBytes(uint8_t *Data,
                                                      size_t Size,
                                                      size_t MaxSize) {
  const size_t kMinBytesToInsert = 3;
  if (Size + kMinBytesToInsert >= MaxSize) return 0;
  size_t MaxBytesToInsert = std::min(MaxSize - Size, (size_t)128);
  size_t N = Rand(MaxBytesToInsert - kMinBytesToInsert + 1) + kMinBytesToInsert;
  assert(Size + N <= MaxSize && N);
  size_t Idx = Rand(Size + 1);
  // Insert new values at Data[Idx].
  memmove(Data + Idx + N, Data + Idx, Size - Idx);
  // Give preference to 0x00 and 0xff.
  uint8_t Byte = Rand.RandBool() ? Rand(256) : (Rand.RandBool() ? 0 : 255);
  for (size_t i = 0; i < N; i++)
    Data[Idx + i] = Byte;
  return Size + N;
}
```

## ChangeByte

将`Data`中一个随机位置处的内容修改为一个随机字符。

```cpp
size_t MutationDispatcher::Mutate_ChangeByte(uint8_t *Data, size_t Size,
                                             size_t MaxSize) {
  if (Size > MaxSize) return 0;
  size_t Idx = Rand(Size);
  Data[Idx] = RandCh(Rand);
  return Size;
}
```

## ChangeBit

反转`Data`中一个随机位置处的数据的随机一位。

```cpp
size_t MutationDispatcher::Mutate_ChangeBit(uint8_t *Data, size_t Size,
                                            size_t MaxSize) {
  if (Size > MaxSize) return 0;
  size_t Idx = Rand(Size);
  Data[Idx] ^= 1 << Rand(8);
  return Size;
}
```

## ShuffleBytes

打乱`Data`中随机位置起`[1,8]`个随机数据的顺序，由`std::shuffle`实现。

```cpp
size_t MutationDispatcher::Mutate_ShuffleBytes(uint8_t *Data, size_t Size,
                                               size_t MaxSize) {
  if (Size > MaxSize || Size == 0) return 0;
  size_t ShuffleAmount =
      Rand(std::min(Size, (size_t)8)) + 1; // [1,8] and <= Size.
  size_t ShuffleStart = Rand(Size - ShuffleAmount);
  assert(ShuffleStart + ShuffleAmount <= Size);
  std::shuffle(Data + ShuffleStart, Data + ShuffleStart + ShuffleAmount, Rand);
  return Size;
}
```

## ChangeASCIIInt

在`Data`中寻找一段连续数字数据`[B,E)`，然后手动计算其数值并随机做一次计算（自增1，自减1，除以2，乘2，取小于其平方的随机数），并将新数值手动解析为字符串放入`[B,E)`中，不足补`0`，超出截断。

```cpp
size_t MutationDispatcher::Mutate_ChangeASCIIInteger(uint8_t *Data, size_t Size,
                                                     size_t MaxSize) {
  if (Size > MaxSize) return 0;
  size_t B = Rand(Size);
  while (B < Size && !isdigit(Data[B])) B++;
  if (B == Size) return 0;
  size_t E = B;
  while (E < Size && isdigit(Data[E])) E++;
  assert(B < E);
  // now we have digits in [B, E).
  // strtol and friends don't accept non-zero-teminated data, parse it manually.
  uint64_t Val = Data[B] - '0';
  for (size_t i = B + 1; i < E; i++)
    Val = Val * 10 + Data[i] - '0';

  // Mutate the integer value.
  switch(Rand(5)) {
    case 0: Val++; break;
    case 1: Val--; break;
    case 2: Val /= 2; break;
    case 3: Val *= 2; break;
    case 4: Val = Rand(Val * Val); break;
    default: assert(0);
  }
  // Just replace the bytes with the new ones, don't bother moving bytes.
  for (size_t i = B; i < E; i++) {
    size_t Idx = E + B - i - 1;
    assert(Idx >= B && Idx < E);
    Data[Idx] = (Val % 10) + '0';
    Val /= 10;
  }
  return Size;
}
```

## ChangeBinInt

选`Data`中随机位置处的随机长度（8、4、2、1字节之一）数据，将`Size`（随机数据位于`Data`前`64`个位置中）或其自身的值随机变换后替换该处数据。

```cpp
size_t MutationDispatcher::Mutate_ChangeBinaryInteger(uint8_t *Data,
                                                      size_t Size,
                                                      size_t MaxSize) {
  if (Size > MaxSize) return 0;
  switch (Rand(4)) {
    case 3: return ChangeBinaryInteger<uint64_t>(Data, Size, Rand);
    case 2: return ChangeBinaryInteger<uint32_t>(Data, Size, Rand);
    case 1: return ChangeBinaryInteger<uint16_t>(Data, Size, Rand);
    case 0: return ChangeBinaryInteger<uint8_t>(Data, Size, Rand);
    default: assert(0);
  }
  return 0;
}

template<class T>
size_t ChangeBinaryInteger(uint8_t *Data, size_t Size, Random &Rand) {
  if (Size < sizeof(T)) return 0;
  size_t Off = Rand(Size - sizeof(T) + 1);
  assert(Off + sizeof(T) <= Size);
  T Val;
  if (Off < 64 && !Rand(4)) {
    Val = Size;
    if (Rand.RandBool())
      Val = Bswap(Val);
  } else {
    memcpy(&Val, Data + Off, sizeof(Val));
    T Add = Rand(21);
    Add -= 10;
    if (Rand.RandBool())
      Val = Bswap(T(Bswap(Val) + Add)); // Add assuming different endiannes.
    else
      Val = Val + Add;               // Add assuming current endiannes.
    if (Add == 0 || Rand.RandBool()) // Maybe negate.
      Val = -Val;
  }
  memcpy(Data + Off, &Val, sizeof(Val));
  return Size;
}

// compiler-rt/lib/fuzzer/FuzzerBuiltins.h
// GNU
inline uint8_t  Bswap(uint8_t x)  { return x; }
inline uint16_t Bswap(uint16_t x) { return __builtin_bswap16(x); } // 按字节顺序翻转，0xaabb -> 0xbbaa
inline uint32_t Bswap(uint32_t x) { return __builtin_bswap32(x); }
inline uint64_t Bswap(uint64_t x) { return __builtin_bswap64(x); }
```

## CopyPart

随机选择`CopyPartOf`或`InsertPartOf`，`CopyPartOf`将`Data`中随机位置的随即长度的数据拷贝到该位置前的随机位置处进行覆写，`InsertPartOf`将`Data`中随机位置的随机长度的数据插入到随机位置中（`ToInsertPos`数据后移）。

```cpp
// Overwrites part of To[0,ToSize) with a part of From[0,FromSize).
// Returns ToSize.
size_t MutationDispatcher::CopyPartOf(const uint8_t *From, size_t FromSize,
                                      uint8_t *To, size_t ToSize) {
  // Copy From[FromBeg, FromBeg + CopySize) into To[ToBeg, ToBeg + CopySize).
  size_t ToBeg = Rand(ToSize);
  size_t CopySize = Rand(ToSize - ToBeg) + 1;
  assert(ToBeg + CopySize <= ToSize);
  CopySize = std::min(CopySize, FromSize);
  size_t FromBeg = Rand(FromSize - CopySize + 1);
  assert(FromBeg + CopySize <= FromSize);
  memmove(To + ToBeg, From + FromBeg, CopySize);
  return ToSize;
}

// Inserts part of From[0,ToSize) into To.
// Returns new size of To on success or 0 on failure.
size_t MutationDispatcher::InsertPartOf(const uint8_t *From, size_t FromSize,
                                        uint8_t *To, size_t ToSize,
                                        size_t MaxToSize) {
  if (ToSize >= MaxToSize) return 0;
  size_t AvailableSpace = MaxToSize - ToSize;
  size_t MaxCopySize = std::min(AvailableSpace, FromSize);
  size_t CopySize = Rand(MaxCopySize) + 1;
  size_t FromBeg = Rand(FromSize - CopySize + 1);
  assert(FromBeg + CopySize <= FromSize);
  size_t ToInsertPos = Rand(ToSize + 1);
  assert(ToInsertPos + CopySize <= MaxToSize);
  size_t TailSize = ToSize - ToInsertPos;
  if (To == From) {
    MutateInPlaceHere.resize(MaxToSize);
    memcpy(MutateInPlaceHere.data(), From + FromBeg, CopySize);
    memmove(To + ToInsertPos + CopySize, To + ToInsertPos, TailSize);
    memmove(To + ToInsertPos, MutateInPlaceHere.data(), CopySize);
  } else {
    memmove(To + ToInsertPos + CopySize, To + ToInsertPos, TailSize);
    memmove(To + ToInsertPos, From + FromBeg, CopySize);
  }
  return ToSize + CopySize;
}

size_t MutationDispatcher::Mutate_CopyPart(uint8_t *Data, size_t Size,
                                           size_t MaxSize) {
  if (Size > MaxSize || Size == 0) return 0;
  // If Size == MaxSize, `InsertPartOf(...)` will
  // fail so there's no point using it in this case.
  if (Size == MaxSize || Rand.RandBool())
    return CopyPartOf(Data, Size, Data, Size);
  else
    return InsertPartOf(Data, Size, Data, Size, MaxSize);
}
```

## CrossOver

从`CrossOver`、`InsertPartOf`和`CopyPartOf`中随机选择一种动作，其中`CrossOver`将`Data`和`CrossOverWith`中的数据以每次轮流取随机大小的块交叉排列组合在一起，`InsertPartOf`将`CrossOverWith`中随机位置处随机长度的数据插入到`Data`的随机位置处，`CopyPartOf`用`CrossOverWith`中随机位置处随机长度的数据对`Data`随机位置处的数据进行覆写。

```cpp
size_t MutationDispatcher::Mutate_CrossOver(uint8_t *Data, size_t Size,
                                            size_t MaxSize) {
  if (Size > MaxSize) return 0;
  if (Size == 0) return 0;
  if (!CrossOverWith) return 0;
  const Unit &O = *CrossOverWith;
  if (O.empty()) return 0;
  MutateInPlaceHere.resize(MaxSize);
  auto &U = MutateInPlaceHere;
  size_t NewSize = 0;
  switch(Rand(3)) {
    case 0:
      NewSize = CrossOver(Data, Size, O.data(), O.size(), U.data(), U.size());
      break;
    case 1:
      NewSize = InsertPartOf(O.data(), O.size(), U.data(), U.size(), MaxSize);
      if (!NewSize)
        NewSize = CopyPartOf(O.data(), O.size(), U.data(), U.size());
      break;
    case 2:
      NewSize = CopyPartOf(O.data(), O.size(), U.data(), U.size());
      break;
    default: assert(0);
  }
  assert(NewSize > 0 && "CrossOver returned empty unit");
  assert(NewSize <= MaxSize && "CrossOver returned overisized unit");
  memcpy(Data, U.data(), NewSize);
  return NewSize;
}

// compiler-rt/lib/fuzzer/FuzzerLoop.cpp
if (Options.DoCrossOver)
  MD.SetCrossOverWith(&Corpus.ChooseUnitToMutate(MD.GetRand()).U);

// compiler-rt/lib/fuzzer/FuzzerMutate.h
void SetCrossOverWith(const Unit *U) { CrossOverWith = U; }

// llvm-project/compiler-rt/lib/fuzzer/FuzzerCrossOver.cpp
// Cross Data1 and Data2, store the result (up to MaxOutSize bytes) in Out.
size_t MutationDispatcher::CrossOver(const uint8_t *Data1, size_t Size1,
                                     const uint8_t *Data2, size_t Size2,
                                     uint8_t *Out, size_t MaxOutSize) {
  assert(Size1 || Size2);
  MaxOutSize = Rand(MaxOutSize) + 1;
  size_t OutPos = 0;
  size_t Pos1 = 0;
  size_t Pos2 = 0;
  size_t *InPos = &Pos1;
  size_t InSize = Size1;
  const uint8_t *Data = Data1;
  bool CurrentlyUsingFirstData = true;
  while (OutPos < MaxOutSize && (Pos1 < Size1 || Pos2 < Size2)) {
    // Merge a part of Data into Out.
    size_t OutSizeLeft = MaxOutSize - OutPos;
    if (*InPos < InSize) {
      size_t InSizeLeft = InSize - *InPos;
      size_t MaxExtraSize = std::min(OutSizeLeft, InSizeLeft);
      size_t ExtraSize = Rand(MaxExtraSize) + 1;
      memcpy(Out + OutPos, Data + *InPos, ExtraSize);
      OutPos += ExtraSize;
      (*InPos) += ExtraSize;
    }
    // Use the other input data on the next iteration.
    InPos  = CurrentlyUsingFirstData ? &Pos2 : &Pos1;
    InSize = CurrentlyUsingFirstData ? Size2 : Size1;
    Data   = CurrentlyUsingFirstData ? Data2 : Data1;
    CurrentlyUsingFirstData = !CurrentlyUsingFirstData;
  }
  return OutPos;
}
```

## ManualDict

在参数`-dict`指定的`ManualDictionary`中随机选择一个`Word`插入或覆写到`Data`的随机位置。

```cpp

// compiler-rt/lib/fuzzer/FuzzerMutate.h
// Dictionary provided by the user via -dict=DICT_FILE.
Dictionary ManualDictionary;

size_t MutationDispatcher::Mutate_AddWordFromManualDictionary(uint8_t *Data,
                                                              size_t Size,
                                                              size_t MaxSize) {
  return AddWordFromDictionary(ManualDictionary, Data, Size, MaxSize);
}

size_t MutationDispatcher::AddWordFromDictionary(Dictionary &D, uint8_t *Data,
                                                 size_t Size, size_t MaxSize) {
  if (Size > MaxSize) return 0;
  if (D.empty()) return 0;
  DictionaryEntry &DE = D[Rand(D.size())];
  Size = ApplyDictionaryEntry(Data, Size, MaxSize, DE);
  if (!Size) return 0;
  DE.IncUseCount();
  CurrentDictionaryEntrySequence.push_back(&DE);
  return Size;
}

size_t MutationDispatcher::ApplyDictionaryEntry(uint8_t *Data, size_t Size,
                                                size_t MaxSize,
                                                DictionaryEntry &DE) {
  const Word &W = DE.GetW();
  bool UsePositionHint = DE.HasPositionHint() &&
                         DE.GetPositionHint() + W.size() < Size &&
                         Rand.RandBool();
  if (Rand.RandBool()) {  // Insert W.
    if (Size + W.size() > MaxSize) return 0;
    size_t Idx = UsePositionHint ? DE.GetPositionHint() : Rand(Size + 1);
    memmove(Data + Idx + W.size(), Data + Idx, Size - Idx);
    memcpy(Data + Idx, W.data(), W.size());
    Size += W.size();
  } else {  // Overwrite some bytes with W.
    if (W.size() > Size) return 0;
    size_t Idx = UsePositionHint ? DE.GetPositionHint() : Rand(Size - W.size());
    memcpy(Data + Idx, W.data(), W.size());
  }
  return Size;
}
```

## PersAutoDict

在`PersistentAutoDictionary`（运行过程中搜集有助于提升覆盖率的数据）中随机选择一个`Word`插入或覆写到`Data`的随机位置。

```cpp

// compiler-rt/lib/fuzzer/FuzzerLoop.cpp
Dictionary PersistentAutoDictionary;

size_t MutationDispatcher::Mutate_AddWordFromPersistentAutoDictionary(
    uint8_t *Data, size_t Size, size_t MaxSize) {
  return AddWordFromDictionary(PersistentAutoDictionary, Data, Size, MaxSize);
}
```

## CMP

`-fsanitize-coverage=trace-cmp (on by default as part of -fsanitize=fuzzer)`

在`TableOfRecentCompares(TORC)`（运行过程中记录执行过的比较操作）中随机选择一个`Word`插入或覆写到`Data`的随机位置。

```cpp
size_t MutationDispatcher::Mutate_AddWordFromTORC(
    uint8_t *Data, size_t Size, size_t MaxSize) {
  Word W;
  DictionaryEntry DE;
  switch (Rand(4)) {
  case 0: {
    auto X = TPC.TORC8.Get(Rand.Rand());
    DE = MakeDictionaryEntryFromCMP(X.A, X.B, Data, Size);
  } break;
  case 1: {
    auto X = TPC.TORC4.Get(Rand.Rand());
    if ((X.A >> 16) == 0 && (X.B >> 16) == 0 && Rand.RandBool())
      DE = MakeDictionaryEntryFromCMP((uint16_t)X.A, (uint16_t)X.B, Data, Size);
    else
      DE = MakeDictionaryEntryFromCMP(X.A, X.B, Data, Size);
  } break;
  case 2: {
    auto X = TPC.TORCW.Get(Rand.Rand());
    DE = MakeDictionaryEntryFromCMP(X.A, X.B, Data, Size);
  } break;
  case 3: if (Options.UseMemmem) {
    auto X = TPC.MMT.Get(Rand.Rand());
    DE = DictionaryEntry(X);
  } break;
  default:
    assert(0);
  }
  if (!DE.GetW().size()) return 0;
  Size = ApplyDictionaryEntry(Data, Size, MaxSize, DE);
  if (!Size) return 0;
  DictionaryEntry &DERef =
      CmpDictionaryEntriesDeque[CmpDictionaryEntriesDequeIdx++ %
                                kCmpDictionaryEntriesDequeSize];
  DERef = DE;
  CurrentDictionaryEntrySequence.push_back(&DERef);
  return Size;
}
```

## Custom

调用用户自定义实现的数据变异方法。

```cpp

// 用户定义方法接口，和LLVMFuzzerTestOneInput位于同一cpp中
// extern "C" size_t LLVMFuzzerCustomMutator

size_t MutationDispatcher::Mutate_Custom(uint8_t *Data, size_t Size,
                                         size_t MaxSize) {
  return EF->LLVMFuzzerCustomMutator(Data, Size, MaxSize, Rand.Rand());
}
```

## CustomCrossOver

调用用户自定义实现的数据交叉方法。

```cpp
size_t MutationDispatcher::Mutate_CustomCrossOver(uint8_t *Data, size_t Size,
                                                  size_t MaxSize) {
  if (Size == 0)
    return 0;
  if (!CrossOverWith) return 0;
  const Unit &Other = *CrossOverWith;
  if (Other.empty())
    return 0;
  CustomCrossOverInPlaceHere.resize(MaxSize);
  auto &U = CustomCrossOverInPlaceHere;
  size_t NewSize = EF->LLVMFuzzerCustomCrossOver(
      Data, Size, Other.data(), Other.size(), U.data(), U.size(), Rand.Rand());
  if (!NewSize)
    return 0;
  assert(NewSize <= MaxSize && "CustomCrossOver returned overisized unit");
  memcpy(Data, U.data(), NewSize);
  return NewSize;
}
```

## 策略选择

`LibFuzzer`随机选择编译策略，其关键代码位于`515`行。

```cpp
// Mutates Data in place, returns new size.
size_t MutationDispatcher::MutateImpl(uint8_t *Data, size_t Size,
                                      size_t MaxSize,
                                      Vector<Mutator> &Mutators) {
  assert(MaxSize > 0);
  // Some mutations may fail (e.g. can't insert more bytes if Size == MaxSize),
  // in which case they will return 0.
  // Try several times before returning un-mutated data.
  for (int Iter = 0; Iter < 100; Iter++) {
    auto M = Mutators[Rand(Mutators.size())]; // 这里
    size_t NewSize = (this->*(M.Fn))(Data, Size, MaxSize);
    if (NewSize && NewSize <= MaxSize) {
      if (Options.OnlyASCII)
        ToASCII(Data, NewSize);
      CurrentMutatorSequence.push_back(M);
      return NewSize;
    }
  }
  *Data = ' ';
  return 1;   // Fallback, should not happen frequently.
}
```

随机选择编译策略的方法明显有所不足，因此有学者对此进行了改进，比如[FuzzerGym: A Competitive Framework for Fuzzing and Learning](https://arxiv.org/abs/1807.07490)，改论文采用强化学习（DQN）的方法来改进fuzz的效率，在此感谢Drozd教授的指导。

## 总结

本文总结介绍了`LibFuzzer`的变异策略，有助于我们对其进行进一步的改进。