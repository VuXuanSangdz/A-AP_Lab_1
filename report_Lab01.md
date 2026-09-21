# Lab 1 — Hướng dẫn chi tiết cho sinh viên mới học xử lý âm thanh

Học phần **CSE457 · Xử lý âm thanh và tiếng nói**  
Bài: *Phân tích và xử lý tín hiệu âm thanh số* (FFT/STFT, lọc số, lượng tử hóa, mã hóa)

Tài liệu này viết theo đúng trình tự **A → G** trong đề Lab. Mỗi bước gồm: *đang làm gì*, *vì sao*, *làm thế nào*, *đọc kết quả ra sao*, và *câu nhận xét có thể đưa vào báo cáo*.

---

## Bản đồ tư duy: Lab này đang đi đường nào?

Hãy tưởng tượng tai bạn nghe một bài nhạc. Máy tính **không nghe**. Máy tính chỉ thấy một dãy số. Cả Lab 1 là hành trình biến “dãy số” thành hiểu biết kỹ thuật:

```
Tệp âm thanh
    → đọc ra mẫu số  (Phần A)
    → nhìn theo thời gian: waveform, RMS  (Phần B)
    → nhìn theo tần số: FFT  (Phần C)
    → nhìn đồng thời thời gian và tần số: STFT  (Phần D–E)
    → thay đổi có chủ đích: lọc FIR  (Phần F)
    → tiết kiệm bit: lượng tử, resample, nén  (Phần G)
```

Quy tắc vàng khi viết nhận xét: **tham số → đồ thị/audio → số liệu → kết luận**. Không viết “nghe hay hơn” nếu không chỉ ra được chỗ nào trên phổ hoặc SNR thay đổi.

---

## 0. Chuẩn bị trước khi gõ code

### Cần mang gì vào buổi Lab

- Python 3, các gói: `numpy`, `scipy`, `matplotlib`.
- (Tuỳ chọn) `pydub` + `ffmpeg` nếu giảng viên bắt đọc MP3 trực tiếp.
- Tai nghe: nhiều kết luận Lab chỉ “chốt” được khi nghe file xuất ra.

Cài nhanh:

```text
pip install -r requirements.txt
```

Mở `Lab01.ipynb` rồi **Run All**. Notebook tự tạo file âm thanh và chạy hết A → G.

```text
pip install -r requirements.txt
```

### File dùng trong bài này

Không dùng file mẫu *Western Cowboy Texas Music* của đề. Tín hiệu được **tự tổng hợp ngay trong notebook** `Lab01.ipynb` (hàm `make_original_track`):

| Tệp | Vai trò |
|-----|---------|
| `audio/input_stereo.wav` | Gốc lossless để phân tích |
| `audio/input_mono.wav` | Trung bình 2 kênh |
| `audio/lab01_original.mp3` | Bản nén ~256 kbps để tính compression ratio |

Cấu trúc 24 giây:

- **0–16 s:** nhạc, hợp âm Am quanh **F0 = 220 Hz (A3)**
- **16–24 s:** tiếng nói tổng hợp, **F0 = 120 Hz** (nguyên âm /a/ rồi /i/)

Đây đúng tinh thần mục 5 đề Lab: một đoạn nhạc và một đoạn tiếng nói, file của mình, không sao chép case study.

---

## Phần A — Đọc file và kiểm tra “giấy khai sinh” của tín hiệu

### A.1. Đang làm gì?

Biến tệp trên đĩa thành mảng NumPy `x[n]`, đồng thời ghi lại:

- $F_s$ (sampling rate, Hz)
- số kênh $C$
- thời lượng
- số bit/mẫu $B$
- kích thước file
- Peak, RMS

### A.2. Vì sao bước này quyết định cả bài?

Tín hiệu vật lý $x_a(t)$ liên tục. Máy tính chỉ giữ

$$
x[n]=x_a(nT),\qquad F_s=1/T.
$$

**Nếu sai $F_s$, mọi trục Hz về sau đều sai.** Ví dụ tưởng $F_s=16000$ trong khi file là 44100 thì đỉnh 147 Hz sẽ bị vẽ thành chỗ khác.

Định lý Nyquist: muốn giữ tần số đến $F_{\max}$ thì $F_s \ge 2F_{\max}$.  
Với CD, $F_s=44100$ Hz ⇒ tần số Nyquist $F_s/2=22050$ Hz. Tai người trưởng thành hiếm khi nghe trên 16–18 kHz, nên 44.1 kHz là đủ cho nhạc.

### A.3. Làm thế nào (từng ý)

1. Đọc WAV PCM 16-bit. Mỗi mẫu là số nguyên $[-32768, 32767]$.
2. Chuẩn hóa về $[-1,1]$:

   $$
   x_{\text{float}}[n]=\frac{x_{\text{int16}}[n]}{2^{15}}=\frac{x_{\text{int16}}[n]}{32768}.
   $$

   Đây đúng công thức đề: chia cho $2^{8\cdot\text{sample_width}-1}$.
3. Nếu stereo, tạo mono bằng trung bình hai kênh:

   $$
   x_{\text{mono}}[n]=\frac{x_L[n]+x_R[n]}{2}.
   $$

   Phân tích phổ trên mono để không “đếm hai lần” cùng một nguồn.
4. Tính Peak và RMS trên mono.
5. Tính tốc độ bit PCM lý thuyết:

   $$
   R_{\text{PCM}}=F_s\cdot B\cdot C \quad [\text{bit/s}].
   $$

   CD stereo: $44100\times 16\times 2=1{,}411{,}200$ bit/s $=1.4112$ Mbps.

### A.4. Số liệu đo được

| Đại lượng | Giá trị |
|-----------|---------|
| $F_s$ | 44 100 Hz |
| Nyquist | 22 050 Hz |
| Kênh | 2 (stereo) |
| Thời lượng | **24.000 s** (1 058 400 mẫu) |
| Bit/mẫu khi lưu PCM | 16 |
| Peak (mono) | 0.802 |
| RMS (mono) | 0.137 (−17.27 dBFS) |
| Clipping | Không (không mẫu nào $\ge 0.999$) |
| WAV stereo | 4.23 MB |
| MP3 | 0.77 MB ≈ 256 kbps |
| $R_{\text{PCM}}$ | 1 411.2 kbps |

`dBFS` nghĩa là so với full-scale $=1$. RMS = 0.137 ⇒ $20\log_{10}(0.137)\approx -17.3$ dBFS: tín hiệu còn headroom, chưa “căng” hết thang.

### A.5. Nhận xét mẫu (2–4 câu)

File đúng chuẩn CD: 44.1 kHz, 16 bit, stereo, **tự tạo 24 s**. Peak < 1 nên quá trình lưu PCM không clip. Nửa đầu là nhạc, nửa sau (16–24 s) là tiếng nói — nhìn waveform đã thấy hai “chất liệu” khác nhau. Kích thước WAV ≈ 4.23 MB khớp $R_{\text{PCM}}\times\text{duration}/8$; MP3 ~256 kbps nhỏ hơn khoảng 5.5 lần chỉ vì **cách mã hóa**.

### A.6. Lỗi hay gặp

- Quên chuẩn hóa: vẽ waveform với trục $\pm 32768$ rồi so với file khác đã chia $[-1,1]$.
- Coi số kênh = 1 trong khi file stereo, tính bit rate thiếu hệ số 2.
- Đọc MP3 bằng thư viện rồi tin rằng đã có tín hiệu gốc.

---

## Phần B — Miền thời gian: waveform, Peak, RMS, năng lượng

### B.1. Đang làm gì?

Nhìn `x[n]` theo trục thời gian. Tính ba đại lượng đề bắt buộc, vẽ toàn tệp và hai đoạn 1 giây có tính chất khác nhau.

$$
\text{Peak}=\max_n|x[n]|,\quad
\text{RMS}=\sqrt{\frac{1}{N}\sum_n x^2[n]},\quad
E=\sum_n x^2[n].
$$

### B.2. Ý nghĩa vật lý từng đại lượng

| Đại lượng | Nói lên điều gì | Không nói lên điều gì |
|-----------|-----------------|------------------------|
| Peak | Có nguy cơ clip không; nốt “đánh mạnh” | Độ to cảm nhận (một spike ngắn Peak cao nhưng RMS thấp) |
| RMS | Mức năng lượng hiệu dụng, gần “độ to” hơn Peak | Tần số / cao độ |
| $E$ | Năng lượng tổng của đoạn | Công bằng nếu hai đoạn **khác độ dài** — lúc đó so RMS |

Clipping: khi $|x[n]|$ bị kẹp ở $\pm 1$, đỉnh sóng bị “cắt phẳng”. Nghe méo, phổ xuất hiện harmonic giả.

### B.3. Cách chọn hai đoạn

Đề yêu cầu hai đoạn khác đặc tính. Ở đây:

- **1.0–2.0 s:** nhạc thưa, Peak = 0.442, RMS = 0.082, $E$ = 295.
- **10.0–11.0 s:** hòa âm dày (đoạn dùng cho FFT), Peak = 0.594, RMS = 0.135, $E$ = 798.

Cùng dài 1 s nên so $E$ là công bằng: đoạn nhạc dày có năng lượng **gấp ~2.7 lần**.

### B.4. Nhận xét mẫu

Waveform toàn bài cho thấy hai khối: nhạc (0–16 s) rồi tiếng nói (16–24 s). Đoạn 10–11 s “đặc” hơn đoạn 1–2 s cả về Peak lẫn RMS — đó là lý do chọn nó cho FFT. Waveform **không** cho biết nốt A hay C; muốn biết tần số phải sang phần C. Không có clipping.

### B.5. Lỗi hay gặp

- Vẽ 93 s với mọi mẫu (4 triệu điểm) làm máy chậm — downsample khi vẽ (`x[::step]`) là được, **không** được downsample dữ liệu phân tích.
- Kết luận “đoạn này nhiều treble” từ waveform: waveform không tách được tần số.

---

## Phần C — FFT: nhìn tín hiệu bằng tần số

### C.1. Đừng FFT cả bài 93 giây

Đây là lỗi số 1 trong đề. DFT/FFT của cả bài trộn **mọi thời điểm** thành một phổ. Bạn biết “có tần số 147 Hz” nhưng không biết nó xuất hiện lúc nào. Đề bắt chọn **một đoạn ổn định 0.5–1 s**.

Công thức:

$$
X[k]=\sum_{n=0}^{N-1}x[n]e^{-j2\pi kn/N},\qquad
f_k=\frac{kF_s}{N_{\mathrm{FFT}}},\qquad
\Delta f=\frac{F_s}{N_{\mathrm{FFT}}}.
$$

Với tín hiệu thực, `np.fft.rfft` chỉ trả về $0\to F_s/2$.

### C.2. Cửa sổ Hamming trước khi FFT

Cắt đoạn 1 s = nhân cửa sổ chữ nhật. Mép đoạn thường không về 0 ⇒ gián đoạn giả ⇒ phổ bị **rò** (spectral leakage). Nhân Hamming:

$$
w[n]=0.54-0.46\cos\frac{2\pi n}{L-1}
$$

để mép hạ dần về 0. (So sánh cửa sổ để dành phần E.)

### C.3. Δf khác “độ phân giải thật”

Gọi $L$ là độ dài frame (số mẫu thật), $N_{\mathrm{FFT}}$ là độ dài FFT (có thể zero-pad).

- **Bin spacing** $\Delta f=F_s/N_{\mathrm{FFT}}$: trục Hz dày hay thưa khi vẽ.
- **True resolution** $\approx F_s/L$: hai sin phải cách nhau khoảng này mới tách được.

Frame 10–11 s: $L=44100$, $F_s/L=1$ Hz.  
Thí nghiệm NFFT:

| NFFT | $\Delta f$ | True resolution |
|------|------------|-----------------|
| 2048 | 21.53 Hz | 1 Hz |
| 8192 | 5.38 Hz | 1 Hz |
| 65536 | 0.673 Hz | 1 Hz |

Tăng NFFT **không** biến 1 Hz thành 0.67 Hz “thật”. Nó chỉ nội suy đường phổ cho mịn. Đây là câu hỏi 2 của báo cáo.

### C.4. Đỉnh phổ đo được (NFFT = 65536, đoạn 10–11 s)

$\approx$ **220.0, 263.8, 330.4, 440.1, 660.1, 880.2 Hz**.

Đọc như nhạc sĩ + kỹ sư (tín hiệu *tự thiết kế* F0 = 220 Hz):

- 220.0 Hz = A3 (F0 đã đặt).
- 263.8 Hz ≈ $1.2\times 220$ — C4 (quãng ba nhỏ).
- 330.4 Hz ≈ $1.5\times 220$ — E4 (quãng năm).
- 440.1, 660.1, 880.2 Hz ≈ $2,3,4\times 220$ — harmonic / octave.

Đó là **cấu trúc điều hòa**, không phải nhiễu ngẫu nhiên. Vì file do mình tạo, đối chiếu đỉnh đo với F0 đặt sẵn là cách kiểm tra FFT có đúng trục Hz hay không.

Phổ tuyến tính (`|X|`) làm harmonic mạnh át harmonic yếu. Phổ dB (`20\log_{10}|X|`) nhìn thấy “bậc thang” harmonic. Trục dB ở đây là **relative dB** (so với chính phổ đó), không phải dBFS.

### C.5. Nhận xét mẫu

Đoạn 10–11 s có phổ vạch: năng lượng nằm ở tần số rời rạc có quan hệ với 220 Hz. NFFT nhỏ (2048) vẫn thấy đỉnh ~220 Hz nhưng vị trí bị khớp bin thô (~215 Hz). NFFT lớn làm đọc Hz chính xác hơn khi **ghi số**, không phải vì tai bỗng tách được nốt sát nhau hơn.

---

## Phần D — STFT / spectrogram: thời gian và tần số cùng lúc

### D.1. Ý tưởng

Cắt tín hiệu thành frame, mỗi frame một FFT, xếp thành ảnh 2D: trục ngang = thời gian, trục dọc = tần số, màu = mức dB.

$$
x_m[n]=x[n]w[n-mH],\qquad
S_{\mathrm{dB}}[m,k]=20\log_{10}(|X[m,k]|+\varepsilon).
$$

$H$ là hop size (bước trượt). Overlap $=L-H$.

### D.2. Cấu hình chuẩn đề cho

Với $F_s=44100$:

- Frame 25 ms → $L=\mathrm{round}(0.025F_s)=1102$ mẫu
- Hop 10 ms → $H=441$ mẫu
- Overlap $=661$ mẫu ≈ **60%**
- `nfft=2048` (zero-pad nhẹ cho trục tần số mịn)

Đoạn vẽ: **8–18 s** (cuối phần nhạc, có transient; sát lúc chuyển sang tiếng nói).

### D.3. Thí nghiệm 10 / 25 / 50 ms — kiểm soát biến

**Chỉ đổi độ dài frame.** Giữ hop = 10 ms và **cùng thang màu** (`vmin`/`vmax` chung). Nếu mỗi ảnh tự scale màu, bạn sẽ “thấy khác” dù năng lượng không đổi — lỗi đề liệt kê.

Cách đọc ảnh:

| Hiện tượng trên spectrogram | Ý nghĩa |
|-----------------------------|---------|
| Vạch ngang | Tần số ổn định theo thời gian (hòa âm, nốt kéo) |
| Tia dọc | Transient (pluck, kick): ngắn theo thời gian, rộng theo tần số |
| Vùng sáng dải thấp | Năng lượng bass/thân đàn |
| Vùng tối dải cao | Ít high-frequency content |

Trade-off quan sát được:

- **10 ms:** tia dọc (nhịp) sắc; vạch harmonic dày, mờ — tần số kém.
- **50 ms:** harmonic như kẻ line; nhịp bị sệt — thời gian kém.
- **25 ms:** điểm cân bằng đề khuyến nghị.

### D.4. Nhận xét mẫu

Năng lượng tập trung dưới ~2 kHz thành dải hài ổn định, đúng với FFT phần C. Các vệt dọc lặp theo nhịp là attack của pluck/trống. Tăng cửa sổ làm “nốt” rõ hơn nhưng “nhịp” mờ hơn: không có cửa sổ tối ưu cho mọi thứ. Zero-padding `nfft=2048` không cứu được frame 10 ms về độ phân giải tần số thật.

---

## Phần E — Rectangular vs Hamming

### E.1. Thí nghiệm công bằng

Cùng đoạn 10–11 s, cùng `NFFT=8192`, chỉ đổi cửa sổ.

### E.2. Đọc đồ thị lobe (rất đáng học)

Đáp ứng tần số của chính cửa sổ:

- **Rectangular:** main-lobe hẹp (tách đỉnh gần nhau tốt) nhưng side-lobe đầu khoảng **−13 dB** — leakage mạnh.
- **Hamming:** main-lobe rộng hơn (~ 2 lần) nên hai sin sát nhau dễ dính; side-lobe khoảng **−40 dB** — đáy phổ sạch.

Trên phổ tín hiệu thật: rectangular có nhiều “gợn” giữa các harmonic; Hamming cô lập đỉnh, đáy thấp hơn.

### E.3. Nhận xét mẫu

Hamming giảm spectral leakage đúng như lý thuyết vì mép frame không còn bước nhảy. Cái giá là đỉnh hơi “mập”. Với nhạc có harmonic cách nhau > 70–100 Hz như bài này, Hamming là lựa chọn đúng: leakage hại hơn việc main-lobe rộng. Nếu đo hai ton cách nhau vài Hz, lúc đó mới cân nhắc cửa sổ hẹp hơn (hoặc frame dài hơn).

---

## Phần F — Lọc số FIR

### F.1. Bộ lọc đang làm gì?

Hệ LTI: $y=x*h$. Trong miền tần số, $Y(f)=H(f)X(f)$: lọc là **nhân phổ với một mặt nạ**.

FIR 201 taps, cửa sổ Hamming:

| Bộ lọc | Cutoff | Việc tai sẽ nghe |
|--------|--------|------------------|
| LPF | 2 kHz | Mất không khí, hi-hat, tiếng “sáng” → muffled |
| HPF | 2 kHz | Mất thân đàn/bass, còn tiếng kim |
| BPF | 400–2000 Hz | Giữ dải giữa (gần formant lời nói / thân guitar) |

Thiết kế: `signal.firwin(numtaps=201, cutoff=..., fs=Fs, window='hamming')`. **Phải truyền `fs`**. Quên `fs` thì cutoff bị hiểu là tần số chuẩn hóa, cả bộ lọc sai.

### F.2. Group delay — bẫy SNR

FIR đối xứng dài $L$ taps trễ

$$
\frac{L-1}{2}=\frac{200}{2}=100\text{ mẫu}=100/44100\approx 2.27\text{ ms}.
$$

Tai hầu như không nhận 2 ms. Nhưng nếu lấy $y[n]-x[n]$ để tính SNR **mà không trượt 100 mẫu**, sai số toàn là lệch pha, SNR giả thấp. Đề liệt kê đúng lỗi này. Khi so phổ, ta so `y[i0+delay : i1+delay]` với `x[i0:i1]`.

### F.3. Khớp H(f) với phổ trước/sau

Đồ thị $|H(f)|$: LPF ≈ 0 dB dưới 2 kHz, sườn cắt, stopband −50 đến −70 dB.  
Phổ đoạn 10–11 s sau LPF: dưới 2 kHz gần như trùng phổ gốc; trên 2 kHz sụt mạnh. Đó là bằng chứng lọc **đúng**, không phải “file nghe nhỏ hơn”.

File nghe thử: `audio/music_lpf_2k.wav`, `audio/filtered_hpf_2k.wav`, `audio/filtered_bpf_400_2000.wav`.

### F.4. Nhận xét mẫu

H(f) thiết kế và phổ đo sau lọc kể cùng một câu chuyện: thành phần > 2 kHz bị suy hao. Nghe LPF, phần nhạc tối hơn vì mất high-frequency content. Độ trễ 2.27 ms không làm méo tembr (pha tuyến tính), chỉ dịch thời gian.

---

## Phần G — Lượng tử hóa, resampling, mã hóa

### G.1. Lượng tử B bit

B bit ⇒ $L=2^B$ mức. Lượng tử đều trên $[-1,1]$:

$$
\Delta=\frac{2}{2^B},\qquad
\hat x[n]=\frac{\mathrm{round}(\mathrm{clip}(x[n],-1,1)\cdot(2^{B-1}-1))}{2^{B-1}-1}.
$$

SNR đo:

$$
\mathrm{SNR}=10\log_{10}\frac{\sum x^2[n]}{\sum(\hat x[n]-x[n])^2}\quad(\text{dB}).
$$

Công thức lý thuyết (Rabiner–Schafer):

$$
\mathrm{SNR}_Q=6B+4.77-20\log_{10}(X_{\max}/\sigma_x).
$$

Với $X_{\max}=1$, $\sigma_x=\mathrm{RMS}=0.138$:

$$
20\log_{10}(1/0.138)\approx 17.19,\qquad
\mathrm{SNR}_Q\approx 6B-12.42.
$$

### G.2. Bảng SNR đo / lý thuyết

| B | $L$ | $\Delta$ | SNR đo (dB) | SNR_Q lý thuyết (dB) |
|---|-----|----------|-------------|----------------------|
| 4 | 16 | 0.125 | 10.50 | 11.58 |
| 6 | 64 | 0.03125 | 23.43 | 23.58 |
| 8 | 256 | 0.0078125 | 35.68 | 35.58 |
| 12 | 4096 | 4.88e-4 | 59.82 | 59.58 |
| 16 | 65536 | 3.05e-5 | 83.91 | 83.58 |

Nhận xét cho câu hỏi “vì sao không đúng tuyệt đối 6 dB/bit”:

- Từ 6→16 bit, mỗi 2 bit ≈ +12 dB, rất gần 6 dB/bit.
- Bit thấp (B=4) lệch hơn vì clip nhẹ / mô hình nhiễu đều $\Delta^2/12$ kém hợp khi bước lượng tử thô.
- Nhạc/thu âm thật (RMS thấp so với Peak, phân bố không Gauss) thường lệch công thức nhiều hơn tín hiệu pedagogic này.

Nghe `audio/quantized_4bit.wav`: nhiễu “rát”, rõ nhất lúc đoạn nhỏ tiếng (quantization noise không bị tín hiệu che). 16 bit gần như trùng gốc với tai thường.

### G.3. Resampling 16 kHz và 8 kHz

Đúng: `scipy.signal.resample_poly` — **có lọc anti-alias**.  
Sai: `x[::k]` — đó là downsample trần, tần số cao gập xuống dải thấp (alias), nghe “kim loại giả”.

Xuống 8 kHz, Nyquist = 4 kHz. Phổ so sánh (upsample lại chỉ để vẽ cùng trục) cho thấy mọi thứ trên 4 kHz biến mất. Tai: giọng/ghi-ta còn nhận ra, mất “air”. Chất lượng giảm vì **mất dải tần**, không vì “ít mẫu thì kém” một cách mơ hồ.

File: `audio/resampled_16000.wav`, `audio/resampled_8000.wav`.

### G.4. Bit rate và compression ratio

$$
R_{\text{PCM}}=44100\times 16\times 2=1{,}411{,}200\ \text{bit/s}.
$$

MP3 đo được ≈ 256.35 kbps, size 769 044 byte.

$$
\text{Compression ratio}=\frac{1411.2}{256}\approx 5.51:1.
$$

Nếu 128 kbps: $1411.2/128\approx 11.03:1$.

Kích thước 24.000 s:

| Định dạng | Size |
|-----------|------|
| PCM 16-bit stereo | 4.23 MB |
| MP3 ~256 kbps | 0.77 MB (tiết kiệm ≈ 81.8%) |
| MP3 128 kbps (ước lượng) | 0.38 MB |

Saving $(\%)=(1-\text{Size}_{\text{nén}}/\text{Size}_{\text{PCM}})\times 100$.

PCM lưu mọi mẫu. MP3 (lossy) lượng tử theo cảm thụ, vứt phần tai bị masking. **Giải nén MP3 ra WAV không hồi sinh thông tin đã mất.**

---

## Bài độc lập (mục 5 đề Lab)

File `lab01_original` **đã gồm cả nhạc và tiếng nói**, đúng gợi ý đề. Có thêm `audio/speech_synthetic.wav` nếu muốn thí nghiệm riêng trên nguyên âm.

Khi thay bằng file thu của mình, giữ quy tắc thí nghiệm có kiểm soát: mỗi lần **chỉ đổi một tham số**.

---

## Trả lời 7 câu hỏi báo cáo

**Câu 1.** $F_s=44.1$ kHz nghĩa là 44100 mẫu/giây. Phổ của tín hiệu rời rạc tuần hoàn chu kỳ $F_s$, hai nửa $0\leftrightarrow F_s/2$ và $F_s/2\leftrightarrow F_s$ đối xứng với tín hiệu thực. Các tần số $>F_s/2$ không có “chỗ riêng”: chúng trùng (alias) với một tần số trong $[0,F_s/2]$. Do đó chỉ biểu diễn độc lập đến 22.05 kHz.

**Câu 2.** Frame 25 ms ⇒ độ dài cửa sổ $L\approx 1102$ mẫu ⇒ độ phân giải vật lý $\approx F_s/L\approx 40$ Hz **không đổi**. NFFT 2048 → 8192 zero-pad: $\Delta f$ nhỏ hơn 4 lần, đường vẽ mịn hơn, **không** thêm thông tin, không tách được hai tần số vốn dính trong main-lobe của cửa sổ 25 ms.

**Câu 3.** Rectangular tương đương cắt đột ngột ⇒ phổ cửa sổ là sinc, side-lobe cao, năng lượng một tone rò sang bin khác (leakage). Hamming làm mép về 0 êm, side-lobe thấp. Đổi lại main-lobe rộng: hai đỉnh gần nhau nằm chung một “nón” nên khó tách — đúng đánh đổi selectivity vs leakage.

**Câu 4.** $(201-1)/2=100$ mẫu. $100/44100\approx 2.27$ ms. Karaoke/live: 2–5 ms bắt đầu cảm giác “lời không dính miệng”; ANC/tai nghe gaming còn khắt khe hơn. Offline (Lab này) 2.27 ms không ảnh hưởng nghe, nhưng **bắt buộc bù** khi so mẫu từng điểm.

**Câu 5.** Số hạng $6B$: mỗi bit nhân đôi số mức, $\Delta$ giảm một nửa, phương sai nhiễu $\Delta^2/12$ giảm 4 lần ⇒ $+6$ dB. Số hạng $-20\log_{10}(X_{\max}/\sigma_x)$: nếu giảm volume đầu vào, $\sigma_x$ nhỏ trong khi thang $X_{\max}$ (full-scale ADC) giữ nguyên, tín hiệu dùng ít mức, SNR lượng tử **giảm**. Mix quá nhỏ rồi tăng volume lúc nghe sẽ kéo theo noise floor.

**Câu 6.** Size PCM $=44100\times 16\times 2\times 60/8=10{,}584{,}000$ byte $=10.584$ MB (theo $10^6$). MP3 128 kbps: $128000\times 60/8=960{,}000$ byte $=0.96$ MB. Ratio $\approx 11.03:1$. WAV trên đĩa còn header ~44 byte, chênh không đáng kể.

**Câu 7.** Hai tình huống nghe tốt hơn nhưng SNR thấp hơn:  
(a) MP3 perceptual: SNR khách quan kém PCM, tai vẫn thấy “gần gốc” vì masking.  
(b) Lọc HPF cắt ù 50 Hz: năng lượng tín hiệu giảm ⇒ SNR có thể giảm, nhưng lời thoại **rõ hơn**.  
SNR đo méo dạng sóng, không đo “dễ nghe”.

---

## Checklist nộp bài (đúng mục 7–8 đề)

Cấu trúc gợi ý:

```text
Lab01_MSSV_HoTen/
├── Lab01.ipynb
├── report_Lab01.md
├── audio/          (input, filtered_*.wav, quantized_*.wav, resampled_*.wav)
└── figures/        (waveform, fft, spectrogram, filter_response, ...)
```

Trước khi nộp:

- [ ] Notebook chạy lại từ đầu, không dán ảnh tay
- [ ] Mỗi hình: tiêu đề, tên trục, đơn vị, ghi tham số (Fs, NFFT, frame, cửa sổ)
- [ ] dB ghi rõ dBFS / relative dB / SNR dB
- [ ] Tên file audio thể hiện cấu hình (`music_lpf_2k.wav`, `quantized_8bit.wav`)
- [ ] Không downsample bằng lấy mẫu thứ $k$
- [ ] Không tăng NFFT rồi viết “độ phân giải vật lý tăng”
- [ ] Không lấy MP3 làm gốc lossless
- [ ] Spectrogram so sánh dùng cùng color scale
- [ ] SNR lọc đã bù group delay nếu có so mẫu

---

## Gợi ý viết 2–4 câu nhận xét (công thức câu)

Sai: “Spectrogram 50 ms đẹp hơn.”  
Đúng: “Với hop 10 ms và cùng thang màu, frame 50 ms làm các vạch harmonic 220–880 Hz mảnh hơn (độ phân giải tần số tốt hơn), đồng thời các tia dọc của nhịp bị loãng trên trục thời gian — đúng trade-off $T\times \Delta f$.”

Sai: “Lọc low-pass nghe ấm.”  
Đúng: “Sau FIR LPF 2 kHz 201 taps, phổ đoạn 10–11 s sụt trên 2 kHz, khớp $|H(f)|$; tai mất dải cao nên âm tối/muffled.”
