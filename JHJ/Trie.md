## 트라이 자료구조

이번주에는 간단하게 `트라이` 알고리즘에 대해서 알아봤습니다. 기본적으로 백준의 [전화번호 목록](https://www.acmicpc.net/problem/5052) 을 풀이하는 것으로 트라이 알고리즘을 학습했습니다.

## 문제의 쟁점

위 링크의 문제는 '해당 번호의 접두어가 있는가' 를 알아봐야 합니다. 예시에서 `911` 긴급번호가 있고 선영이의 번호는 `911-12-54-26` 이었습니다. 이런 경우는 911로 연결되기에 NO를 출력해야 하는 문제입니다.

만일 이를 완전 탐색으로 보게 된다면 모든 문자열을 비교해야 하기에 엄청난 시간이 걸리게 될 것입니다. 테스트케이스가 50개 전화번호수도 10,000개이고 길이도 최대 10자리이므로 완전 탐색으로는 이를 찾아내는데 시간이 오래 걸리게 됩니다.

때문에 이번에 나오는 `트라이` 알고리즘을 사용하는 것입니다.


## 트라이 자료구조란

`트라이(Trie)` 는 접두어를 찾는데 특화된 자료구조입니다. 트리의 구조를 따르게 되며 각 노드별로 문자를 1개씩 가져가는 형태입니다.

예를 들어 위 문제에서 911을 입력하면 루트에서 9 - 1 - 1 순으로 노드가 나오게 되는 것입니다. C++ 코드로 보면 아래와 같은 형태를 띄게 됩니다.

```cpp
struct Trie
{
    bool bIsTerminal;    // 해당 노드가 마지막인지 기록 (911의 경우에는 1에서 true)
    Trie* Children[10];  // 위 문제 특수형으로 vector나 map 등을 사용해도 됩니다.

    Trie()
    : bIsTerminal(false), Children()
    {}

    ~Trie()
    {
        delete [] Children;
    }

    void Insert(const string& InputNumber)
    {
        Trie* CurTrie = this;

        for(int i = 0; i < InputNumber.size(); i++)
        {
            // 들어온 전화번호 수만큼 아래 자식 노드들을 늘려 간다
            if(CurTrie->Children[InputNumber[i] - '0'] == nullptr)
            {
                CurTrie->Children[InputNumber[i] - '0'] = new Trie();        
            }
            CurTrie = CurTrie->Children[InputNumber[i] - '0'];

            if(i == InputNumber.size() - 1)
            {
                // 마지막 노드에서는 표시를 해 준다
                CurTrie->bIsTerminal = true;
            }
        }
    }

    bool IsTherePrefix(const string& InputNumber)
    {
        Trie* CurTrie = this;

        for(int i = 0; i < InputNumber.size(); i++)
        {
            if(CurTrie->Children[InputNumber[i] - '0'] != nullptr)
            {
                CurTrie = CurTrie->Children[InputNumber[i] - '0'];
                // 마지막 노드로 표시된게 있다면 true
                if(CurTrie->bIsTerminal)
                {
                    return true;
                }
            }
            else
            {
                return false;
            }
        }
        return false;
    }
};

int main()
{
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);
    cout.tie(nullptr);
    
    int TestCaseNum, PhoneNumberCount;

    cin >> TestCaseNum;

    while (TestCaseNum--)
    {
        cin >> PhoneNumberCount;
        vector<string> NumberList(PhoneNumberCount);
        for(int i = 0; i < PhoneNumberCount; i++)
        {
            cin >> NumberList[i];
        }
        // 정렬을 하고 시작해야 하는 문제
        sort(NumberList.begin(), NumberList.end());

        Trie* NumberTrie = new Trie();
        bool bNotConsistency = true;
        
        for(auto PhoneNumber : NumberList)
        {
            if(NumberTrie->IsTherePrefix(PhoneNumber))
            {
                bNotConsistency = false;
                break;
            }
            NumberTrie->Insert(PhoneNumber);
        }

        if(bNotConsistency)
        {
            cout << "YES\n";
        }
        else
        {
            cout << "NO\n";
        }
        
        delete NumberTrie;
    }
}
```

이런 식으로 문자열의 각 문자를 노드로 관리하기에 접두사의 유무를 찾아내기 좋은 알고리즘 중 하나입니다. L길이의 문자에 대해 접두사가 있는지 찾기 위해서는 `O(L)` 시간만 소요되기 때문이죠.

다만 문자 하나씩 노드가 생성되게 되므로 메모리적으로는 좋지 않은 구조라고 볼 수 있습니다.