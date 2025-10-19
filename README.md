# FaustDSP

```c++
// FaustDSP.hpp
#pragma once
#include <string>
#include <iostream>

// Required Faust headers
#include <faust/dsp/llvm-dsp.h>   // defines llvm_dsp_factory
#include <faust/dsp/dsp.h>        // defines dsp, FAUSTFLOAT, etc.
#include <faust/dsp/libfaust.h>   // for create/delete factory functions

class FaustBaseDSP {
protected:
    llvm_dsp_factory* factory = nullptr;
    dsp* dspInstance = nullptr;
    int sampleRate = 44100;

public:
    FaustBaseDSP(const std::string& name, const std::string& faustCode, int sr = 44100)
        : sampleRate(sr)
    {
        std::string errorMsg;

        factory = createDSPFactoryFromString(
            name, faustCode,
            0, nullptr,
            "",             // target: auto-select (LLVM)
            errorMsg,       // compiler log
            3               // optimization level
        );

        if (!factory) {
            std::cerr << "[Faust] Factory creation failed for " << name
                    << "\nError: " << errorMsg << std::endl;
            return;
        }

        dspInstance = factory->createDSPInstance();
        if (!dspInstance) {
            std::cerr << "[Faust] DSP instance creation failed\n";
            deleteDSPFactory(factory);
            factory = nullptr;
            return;
        }

        dspInstance->init(sampleRate);
    }

    virtual ~FaustBaseDSP() {
        if (dspInstance) delete dspInstance;
        if (factory) deleteDSPFactory(factory);
    }

    bool isValid() const { return dspInstance != nullptr; }

    int getNumInputs() const { return dspInstance ? dspInstance->getNumInputs() : 0; }
    int getNumOutputs() const { return dspInstance ? dspInstance->getNumOutputs() : 0; }
    int getSampleRate() const { return sampleRate; }

    dsp* getDSP() const { return dspInstance; }
};

// ----------- MONO DSP WRAPPER -----------
class FaustMonoDSP : public FaustBaseDSP {
public:
    FaustMonoDSP(const std::string& code, int sr = 44100)
        : FaustBaseDSP("MonoDSP", code, sr)
    {
        if (isValid() && (getNumInputs() != 1 || getNumOutputs() != 1)) {
            std::cerr << "[Faust] Warning: DSP is not mono (" 
                      << getNumInputs() << " in, " << getNumOutputs() << " out)" << std::endl;
        }
    }

    void process(float* input, float* output, int nframes) {
        if (!isValid()) return;

        FAUSTFLOAT* in[1]  = { input };
        FAUSTFLOAT* out[1] = { output };
        dspInstance->compute(nframes, in, out);
    }
};

// ----------- STEREO DSP WRAPPER -----------
class FaustStereoDSP : public FaustBaseDSP {
public:
    FaustStereoDSP(const std::string& code, int sr = 44100)
        : FaustBaseDSP("StereoDSP", code, sr)
    {
        if (isValid() && (getNumInputs() != 2 || getNumOutputs() != 2)) {
            std::cerr << "[Faust] Warning: DSP is not stereo ("
                      << getNumInputs() << " in, " << getNumOutputs() << " out)" << std::endl;
        }
    }

    void process(float* leftIn, float* rightIn,
                 float* leftOut, float* rightOut,
                 int nframes) {
        if (!isValid()) return;

        FAUSTFLOAT* in[2]  = { leftIn, rightIn };
        FAUSTFLOAT* out[2] = { leftOut, rightOut };
        dspInstance->compute(nframes, in, out);
    }
};

```

```
// FaustDSP.cpp
#include "FaustDSP.hpp"
#include <vector>

int main() {
    // Example Faust code
    std::string monoCode = R"(
        import("stdfaust.lib");
        process = fi.lowpass(3, 800) : *(0.8);
    )";

    FaustMonoDSP monoDSP(monoCode, 48000);

    if (!monoDSP.isValid()) return 1;

    const int N = 64;
    std::vector<float> input(N, 0.1f);
    std::vector<float> output(N);

    monoDSP.process(input.data(), output.data(), N);
    std::cout << "Output[0] = " << output[0] << std::endl;

    // Stereo example
    std::string stereoCode = R"(
        import("stdfaust.lib");
        process = _,_ : fi.lowpass(3, 1000);
    )";

    FaustStereoDSP stereoDSP(stereoCode, 48000);

    std::vector<float> L_in(N, 0.2f), R_in(N, 0.2f);
    std::vector<float> L_out(N), R_out(N);

    stereoDSP.process(L_in.data(), R_in.data(), L_out.data(), R_out.data(), N);
    std::cout << "Stereo output[0] = " << L_out[0] << ", " << R_out[0] << std::endl;
}
```

# Bindings

Beautiful — now you’re thinking like a true engine designer. ⚙️

Exactly: once your Faust C++ wrapper works, the next logical step is to **expose it to Python** so you can do dynamic DSP prototyping, live control, or even use it inside something like Jupyter, MaxPy, or Blender.

You have two great options:

---

## 🧩 Option 1 — **Pybind11 (modern and elegant)**

This is the best route today — C++17 native, header-only, and works seamlessly with your existing wrapper classes (`FaustMonoDSP`, `FaustStereoDSP`).

### 🔧 Install pybind11

```bash
sudo apt install pybind11-dev
```

(or `pip install pybind11` if you’ll embed via pip)

---

### 🧠 Example: `faust_bind.cpp`

Assume you already have `FaustDSP.hpp` working.
Then your binding could look like this:

```cpp
#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>
#include "FaustDSP.hpp"

namespace py = pybind11;

PYBIND11_MODULE(pyfaust, m) {
    m.doc() = "Python bindings for embedded Faust DSP via libfaust";

    py::class_<FaustMonoDSP>(m, "FaustMonoDSP")
        .def(py::init<const std::string&, int>(),
             py::arg("faust_code"), py::arg("sample_rate") = 44100)
        .def("is_valid", &FaustMonoDSP::isValid)
        .def("process", [](FaustMonoDSP &self,
                           py::array_t<float> input,
                           py::array_t<float> output) {
            py::buffer_info inbuf = input.request();
            py::buffer_info outbuf = output.request();

            if (inbuf.size != outbuf.size)
                throw std::runtime_error("Input/output size mismatch");

            self.process(static_cast<float*>(inbuf.ptr),
                         static_cast<float*>(outbuf.ptr),
                         (int)inbuf.size);
        });

    py::class_<FaustStereoDSP>(m, "FaustStereoDSP")
        .def(py::init<const std::string&, int>(),
             py::arg("faust_code"), py::arg("sample_rate") = 44100)
        .def("is_valid", &FaustStereoDSP::isValid)
        .def("process", [](FaustStereoDSP &self,
                           py::array_t<float> left_in,
                           py::array_t<float> right_in,
                           py::array_t<float> left_out,
                           py::array_t<float> right_out) {
            py::buffer_info lin = left_in.request();
            py::buffer_info rin = right_in.request();
            py::buffer_info lout = left_out.request();
            py::buffer_info rout = right_out.request();

            if (lin.size != rin.size || lout.size != rout.size)
                throw std::runtime_error("Buffer size mismatch");

            self.process(static_cast<float*>(lin.ptr),
                         static_cast<float*>(rin.ptr),
                         static_cast<float*>(lout.ptr),
                         static_cast<float*>(rout.ptr),
                         (int)lin.size);
        });
}
```

---

### ⚙️ Build the Python module

```bash
c++ -O3 -Wall -shared -std=c++17 -fPIC \
    $(python3 -m pybind11 --includes) \
    -I/usr/local/include/faust \
    -L/usr/local/lib \
    FaustDSP.cpp faust_bind.cpp \
    -lfaust -ldl -lpthread \
    -o pyfaust$(python3-config --extension-suffix)
```

This creates a shared module like `pyfaust.cpython-311-x86_64-linux-gnu.so`.

---

### 🐍 Use it in Python

```python
import numpy as np
import pyfaust

code = """
import("stdfaust.lib");
process = fi.lowpass(3, 800) : *(0.8);
"""

dsp = pyfaust.FaustMonoDSP(code, 48000)
x = np.ones(64, dtype=np.float32) * 0.1
y = np.zeros_like(x)
dsp.process(x, y)
print(y[:5])
```

Boom 💥 — now you’re processing audio through Faust *from Python*.

---

## 🧠 Option 2 — SWIG (old-school, more verbose)

If you need compatibility with older toolchains or languages other than Python, SWIG can wrap it too — but it’s heavier and less Pythonic.

You’d create a `.i` interface file that `%include "FaustDSP.hpp"`, then build with:

```bash
swig -c++ -python faust_bind.i
```

But for modern work (especially if you’re mixing NumPy or real-time DSP prototyping), **Pybind11 is far superior**.

---

## ✅ Summary

| Method       | Best for       | Pros                       | Cons                      |
| ------------ | -------------- | -------------------------- | ------------------------- |
| **Pybind11** | Python 3       | Clean, modern, NumPy-ready | C++17 required            |
| **SWIG**     | Multi-language | Wider reach                | Clunky Python integration |

---

If you tell me your **goal for the Python side** (e.g. prototyping synths, live audio streaming, ML-based modulation, etc.), I can generate a tailored binding layout — including buffer-size flexibility and real-time performance hints.

Would you like me to make that next?
