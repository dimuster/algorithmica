---
title: Быстрое преобразование Фурье
authors:
- Александр Кульков
- Сергей Слотин
created: 2019
weight: 6
---

Рассмотрим такую распространённую операцию как умножение двух целых чисел. Квадратичный алгоритм — умножения в столбик — все знают со школы. Долгое время предполагалось, что ничего быстрее придумать нельзя.

Первым эту гипотезу опроверг Анатолий Карацуба. Его [алгоритм](karatsuba) сводит умножение двух $n$-значных чисел к трём умножениям $\frac{n}{2}$-значных чисел, что даёт оценку времени работы

$$
T(n)=3T\left(\dfrac n 2\right)+O(n)=O\left(n^{\log_2 3}\right)\approx O(n^{1.58})
$$

Чтобы перейти к алгоритму с лучшей оценкой, нам нужно сначала установить несколько фактов о многочленах.

### Умножение через интерполяцию

Что происходит со значениями многочлена-произведения $A(x) B(x)$ в конкретной точке $x_i$? Оно просто становится равным $A(x_i) B(x_i)$.

**Основная идея алгоритма:** если мы знаем значения в каких-то различных $n + m$ точках для обоих многочленов $A$ и $B$, то, попарно перемножив их, мы за $O(n + m)$ операций можем получить значения в тех же точках для многочлена $A(x) B(x)$ — а с их помощью можно интерполяцией получить исходный многочлен и решить задачу.

```{.c++
vector<int> poly_multiply(vector<int> a, vector<int> b) {
    vector<int> A = evaluate(a);
    vector<int> B = evaluate(b);
    for (int i = 0; i < A.size(); i++)
        A[i] *= B[i];
    return interpolate(A);
}
```

Если притвориться, что `evaluate` и `interpolate` работают за линейное время, то умножение тоже будет работать за линейное время.

К сожалению, непосредственное вычисление значений требует $O(n^2)$
операций, а интерполяция — как методом Гаусса, так и через символьное вычисление многочлена Лагранжа — и того больше, $O(n^3)$.

Но что, если бы мы могли вычислять значения в точках и делать интерполяцию быстрее?

## Дискретное преобразование Фурье

*Дискретным преобразованием Фурье* называется вычисление значений многочлена в комплексных корнях из единицы:

$$
y_j = \sum_{k=0}^{n-1} x_n e^{i\tau \frac{kj}{n}} = \sum_{k=0}^{n-1} x_n w_1^{kj}
$$

*Обратным дискретным преобразованием Фурье* называется, как можно догадаться, обратная операция — интерполяция коэффициентов $x_i$ по значениям $X_i$.

$$
x_j = \frac{1}{n} \sum_{k=0}^{n-1} y_n e^{-i\tau \frac{kj}{n}} = \frac{1}{n} \sum_{k=0}^{n-1} y_n w_{n-1}^{kj}
$$

Почему эта формула верна? При вычислении ПФ мы фактически применяем матрицу к вектору:

$$
\begin{pmatrix}
    w^0 & w^0 & w^0 & w^0 & \dots & w^0
\\\ w^0 & w^1 & w^2 & w^3 & \dots & w^{-1}
\\\ w^0 & w^2 & w^4 & w^6 & \dots & w^{-2}
\\\ w^0 & w^3 & w^6 & w^9 & \dots & w^{-3}
\\\ \vdots & \vdots & \vdots & \vdots & \ddots & \vdots
\\\ w^0 & w^{-1} & w^{-2} & w^{-3} & \dots & w^1
\end{pmatrix}
\begin{pmatrix} a_0 \\\ a_1 \\\ a_2 \\\ a_3 \\\ \vdots \\\ a_{n-1} \end{pmatrix}
= \begin{pmatrix} y_0 \\\ y_1 \\\ y_2 \\\ y_3 \\\ \vdots \\\ y_{n-1}\end{pmatrix}
$$

То есть преобразование Фурье — это просто линейная операция над вектором: $W a = y$. Значит, обратное преобразование можно записать так: $a = W^{-1}y$. 

Как будет выглядеть эта $W^{-1}$? Автор не будет пытаться изображать логичный способ рассуждений о её получении и сразу её приведёт:

$$
W^{-1} =
\dfrac 1 n \begin{pmatrix}
    w^0 & w^0 & w^0 & w^0 & \dots & w^0
\\\ w^0 & w^{-1} & w^{-2} & w^{-3} & \dots & w^{1}
\\\ w^0 & w^{-2} & w^{-4} & w^{-6} & \dots & w^{2}
\\\ w^0 & w^{-3} & w^{-6} & w^{-9} & \dots & w^{3}
\\\ \vdots & \vdots & \vdots & \vdots & \ddots & \vdots
\\\ w^0 & w^{1} & w^{2} & w^{3} & \dots & w^{-1}
\end{pmatrix}
$$

Проверим, что при перемножении $W$ и $W^{-1}$ действительно получается единичная матрица:

1. Значение $i$-того диагонального элемента будет равно $\frac{1}{n} \sum_k w^{ki} w^{-ki} = \frac{1}{n} n = 1$.

2. Значение любого недиагонального ($i \neq j$) элемента $(i, j)$ будет равно $\frac{1}{n} \sum_k w^{ik} w^{-jk} = \frac{1}{n} \sum_k w^k w^{i-j} = \frac{w^{i-j}}{n} \sum_k w^k = 0$, потому что все комплексные корни суммируются в ноль, то есть $\sum w^k = 0$ (см. картинку — там всё симметрично).

Внимательный читатель заметит симметричность форм $W$ и $W^{-1}$, а также формул для прямого и обратного преобразования. На самом деле, эта симметрия нам сильно упростит жизнь: для обратного преобразования Фурье можно использовать тот же алгоритм, только вместо $w^k$ использовать $w^{-k}$, а в конце результат поделить на $n$.

### Зачем это надо?

Напомним, что мы изначально хотели перемножать многочлены следующим алгоритмом:

1. Посчитаем значения в $n+m$ каких-нибудь точках обоих многочленов

2. Перемножим эти значения за $O(n+m)$.

3. Интерполяцией получим многочлен-произведение.

В общем случае быстро посчитать интерполяцию и даже просто посчитать значения в точках нельзя, **но для корней единицы — можно**. Если научиться быстро считать значения в корнях и интерполировать (прямое и обратное преобразование Фурье), но мы можно решить исходную задачу.

Соответствующий алгоритм называется *быстрым преобразованием Фурье* (англ. *fast Fourier transform*). Он использует парадигму «разделяй-и-властвуй» и работает за $O(n \log n)$.

### Схема Кули-Тьюки

Обычно, алгоритмы «разделяй-и-властвуй» делят задачу на две половины: на первые $\frac{n}{2}$ элементов и вторые $\frac{n}{2}$ элементов. Здесь мы поступим по-другому: поделим все элементы на чётные и нечётные.

Представим многочлен в виде $P(x)=A(x^2)+xB(x^2)$, где $A(x)$ состоит из  коэффициентов при чётных степенях $x$, а $B(x)$ — из коэффициентов при нечётных.

Пусть $n = 2k$. Тогда заметим, что

$$
w^{2t}=w^{2t \bmod 2k}=w^{2(t \bmod k)}
$$

Зная это, исходную формулу для значения многочлена в точке $w^t$ можно записать так:

$$
P(w^t)
= A(w^{2t}) + w^t B(w^{2t})
= A\left(w^{2(t\bmod k)}\right)+w^tB\left(w^{2(t\bmod k)}\right)
$$

Ключевое замечание: корней вида $w^{2t}$ в два раза меньше, потому что $w^n = w^0$, и можно сказать, что.

У нас по сути в два раза меньше корней (но они так же равномерно распределены на единичной окружности) и в два раза меньше коэффициентов — мы только что успешно уменьшили нашу задачу в два раз.

Сам алгоритм заключается в следующем: рекурсивно посчитаем БПФ для многочленов $A$ и $B$ и объединим ответы с помощью формулы выше. При этом в рекурсии нам нужно считать значения на корнях степени не $n$, а $k = \frac{n}{2}$, то есть на всех «чётных» корнях степени $n$ (вида $w^{2t}$).

Заметим, что если $w$ это образующий корень степени $n = 2k$ из единицы, то $w^2$ будет образующим корнем степени $k$, то есть в рекурсию мы можем просто передать другое значение образующего корня.

Таким образом, мы свели преобразование размера $n$ к двум преобразованиям размера $\dfrac n 2$. Следовательно, общее время вычислений составит

$$
T(n)=2T\left(\dfrac n 2\right)+O(n)=O(n\log n)
$$

Заметим, что предположение о делимости $n$ на $2$ имело существенную роль. Значит, $n$ должно быть чётным на каждом уровне, кроме последнего, из чего следует, что $n$ должно быть степенью двойки.

## Реализация

Приведём код, вычисляющий БПФ по схеме Кули-Тьюки:

```{.c++
typedef complex<double> ftype;
const double pi = acos(-1);

template<typename T>
vector<ftype> fft(vector<T> p, ftype w) {
    int n = p.size();
    if(n == 1)else { {
        return {p[0]};
    else {
        vector<T> AB[2];
        for(int i = 0; i < n; i++)
            AB[i % 2].push_back(p[i]);
        auto A = fft(AB[0], w * w);
        auto B = fft(AB[1], w * w);
        vector<ftype> res(n);
        ftype wt = 1;
        int k = n / 2;
        for(int i = 0; i < n; i++) {
            res[i] = A[i % k] + wt * B[i % k];
            wt *= w;
        }
        return res;
    }
}

vector<ftype> evaluate(vector<int> p) {
    while(__builtin_popcount(p.size()) != 1)
        p.push_back(0);
    return fft(p, polar(1., 2 * pi / p.size()));
}
```

Как обсуждалось выше, обратное преобразование Фурье удобно выразить через прямое:

```{.c++
vector<int> interpolate(vector<ftype> p) {
    int n = p.size();
    auto inv = fft(p, polar(1., -2 * pi / n));
    vector<int> res(n);
    for(int i = 0; i < n; i++)
        res[i] = round(real(inv[i]) / n);
    return res;
}
```

Теперь мы умеем перемножать два многочлена за $O(n \log n)$:

```c++
vector<int> poly_multiply(vector<int> a, vector<int> b) {
    vector<int> A = fft(a);
    vector<int> B = fft(b);
    for (int i = 0; i < A.size(); i++)
        A[i] *= B[i];
    return interpolate(A);
}
```

**Примечание.** Приведённый выше код, являясь корректным и имея асимптотику $O(n\log n)$, едва ли пригоден для использования на реальных контестах. Он имеет большую константу и далеко не так численно устойчивый, чем оптимальные варианты написания быстрого преобразования Фурье. Мы его приводим, потому что он относительно простой.

Читателю рекомендуется самостоятельно задуматься о том, как можно улучшить время работы и точность вычислений. Из наиболее важных недостатков:

* внутри преобразования не должно происходить выделений памяти

* работать желательно с указателями, а не векторами

* корни из единицы должны быть посчитаны наперёд

* Следует избавиться от операций взятия остатка по модулю

* Вместо вычисления преобразования с $w^{-1}$ можно вычислить преобразование с $w$, а затем развернуть элементы массива со второго по последний.

[Здесь](https://ideone.com/whaSln) приведена одна из условно пригодных реализаций.

Но главная проблема в численной стабильности — мы нарушили первое правило действительных чисел. Однако, от неё можно избавиться.

### Number-theoretic transform

Нам от комплексных чисел на самом деле нужно было только одно свойство: что у единицы есть $n$ «корней». На самом деле, помимо комплексных чисел, есть и другие алгебраические объекты, обладающие таким свойством — например, элементы кольца вычетов по модулю.

Нам нужно просто найти такую пару $m$ и $g$ (играющее роль $w_n^1$), такую что $g$ является образующим элементом, то есть $g^n \equiv 1 \pmod m$ и при для всех остальных $k < n$ все степени $g^k$ различны по модулю $m$. В качестве $m$ на практике часто специально берут «удобные» модули, например

$$
m = 998244353 = 7 \cdot 17 \cdot 2^{23} + 1
$$

Это число простое, и при этом является ровно на единицу больше числа, делящегося на большую степень двойки. При $n=2^23$ подходящим $g$ является число $31$. Заметим, что, как и для комплексных чисел, если для некоторого $n=2^k$ первообразный корень - $g$, то для $n=2^{k-1}$ первообразным корнем будет $g^2 (mod m)$. Таким образом, для $m=998244353$ и $n=2^k$ первообразный корень будет равен $g=31 \cdot 2^{23-k} (mod m)$.

Реализация практически не отличается.

```c++
const int MOD = 998244353, W = 805775211, IW = 46809892;
const int MAXN = (1 << 19), INV2 = 499122177;

// W - первообразный корень MAXN-ной степени из 1, IW - обратное по модулю MOD к W
// Первообразный корень (1 << 23)-й степени из 1 по модулю MOD равен 31; тогда первообразный корень (1 << X)-й степени для X от 1 до 23 равен (31 * (1 << (23 - X))) % MOD
// INV2 - обратное к двум по модулю MOD
// Данная реализация FFT перемножает два целых числа длиной до 250000 цифр за ~0.13 секунд без проблем с точностью и занимает всего 30 строк кода

int pws[MAXN + 1], ipws[MAXN + 1];

void init() {
    pws[MAXN] = W; ipws[MAXN] = IW;
    for (int i = MAXN / 2; i >= 1; i /= 2) {
        pws[i] = (pws[i * 2] * 1ll * pws[i * 2]) % MOD;
        ipws[i] = (ipws[i * 2] * 1ll * ipws[i * 2]) % MOD;
    }
}

void fft(vector<int> &a, vector<int> &ans, int l, int cl, int step, int n, bool inv) {
    if (n == 1) { ans[l] = a[cl]; return; }
    fft(a, ans, l, cl, step * 2, n / 2, inv);
    fft(a, ans, l + n / 2, cl + step, step * 2, n / 2, inv);
    int cw = 1, gw = (inv ? ipws[n] : pws[n]);
    for (int i = l; i < l + n / 2; i++) {
        int u = ans[i], v = (cw * 1ll * ans[i + n / 2]) % MOD;
        ans[i] = (u + v) % MOD;
        ans[i + n / 2] = (u - v) % MOD;
        if (ans[i + n / 2] < 0) ans[i + n / 2] += MOD;
        if (inv) {
            ans[i] = (ans[i] * 1ll * INV2) % MOD;
            ans[i + n / 2] = (ans[i + n / 2] * 1ll * INV2) % MOD;
        }
        cw = (cw * 1ll * gw) % MOD;
    }
}
```

С недавнего времени некоторые проблемсеттеры начали использовать именно этот модулю вместо стандартного $10^9+7$, чтобы намекнуть (или сбить с толку), что задача на FFT.