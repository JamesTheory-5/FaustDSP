# FaustDSP

Excellent — let’s do this *right*. 👏
Below is a **complete, modern, production-ready version** of your `FaustDSP.hpp` wrapper, with:

✅ LLVM JIT embedding
✅ Mono & Stereo process paths
✅ Automatic UI parameter collection (knobs/sliders/buttons)
✅ Simple `getParam()` / `setParam()` APIs
✅ Ready for Pybind11 export later

---

# 🎛️ `FaustDSP.hpp` (Full Unified Version)

```cpp
#pragma once
#include <string>
#include <vector>
#include <unordered_map>
#include <iostream>

// Faust core headers
#include <faust/dsp/llvm-dsp.h>
#include <faust/dsp/dsp.h>
#include <faust/dsp/libfaust.h>
#include <faust/gui/UI.h>

// -------------------- Parameter Collector --------------------
struct FaustParam {
    std::string label;
    std::string path;
    FAUSTFLOAT* zone;
    FAUSTFLOAT init;
    FAUSTFLOAT min;
    FAUSTFLOAT max;
    FAUSTFLOAT step;
    std::string type;
};

class FaustParamCollector : public UI {
public:
    std::vector<FaustParam> params;
    std::unordered_map<std::string, FAUSTFLOAT*> lookup;

    void addButton(const char* label, FAUSTFLOAT* zone) override {
        addParam(label, zone, 0, 0, 1, 1, "button");
    }

    void addCheckButton(const char* label, FAUSTFLOAT* zone) override {
        addParam(label, zone, 0, 0, 1, 1, "check");
    }

    void addVerticalSlider(const char* label, FAUSTFLOAT* zone,
                           FAUSTFLOAT init, FAUSTFLOAT min,
                           FAUSTFLOAT max, FAUSTFLOAT step) override {
        addParam(label, zone, init, min, max, step, "vslider");
    }

    void addHorizontalSlider(const char* label, FAUSTFLOAT* zone,
                             FAUSTFLOAT init, FAUSTFLOAT min,
                             FAUSTFLOAT max, FAUSTFLOAT step) override {
        addParam(label, zone, init, min, max, step, "hslider");
    }

    void addNumEntry(const char* label, FAUSTFLOAT* zone,
                     FAUSTFLOAT init, FAUSTFLOAT min,
                     FAUSTFLOAT max, FAUSTFLOAT step) override {
        addParam(label, zone, init, min, max, step, "nentry");
    }

    void openTabBox(const char* label) override {}
    void openHorizontalBox(const char* label) override {}
    void openVerticalBox(const char* label) override {}
    void closeBox() override {}
    void addSoundfile(const char*, const char*, Soundfile**) override {}
    void declare(FAUSTFLOAT*, const char*, const char*) override {}

    void addParam(const std::string& label, FAUSTFLOAT* zone,
                  FAUSTFLOAT init, FAUSTFLOAT min,
                  FAUSTFLOAT max, FAUSTFLOAT step,
                  const std::string& type)
    {
        std::string path = "/" + label;
        params.push_back({label, path, zone, init, min, max, step, type});
        lookup[path] = zone;
    }

    bool setParam(const std::string& path, FAUSTFLOAT val) {
        if (lookup.count(path)) {
            *(lookup[path]) = val;
            return true;
        }
        return false;
    }

    FAUSTFLOAT getParam(const std::string& path) const {
        if (lookup.count(path)) {
            return *(lookup.at(path));
        }
        return 0;
    }
};

// -------------------- Base DSP Wrapper --------------------
class FaustBaseDSP {
protected:
    llvm_dsp_factory* factory = nullptr;
    dsp* dspInstance = nullptr;
    int sampleRate = 44100;

public:
    std::vector<FaustParam> params;
    std::unordered_map<std::string, FAUSTFLOAT*> paramLookup;

    FaustBaseDSP(const std::string& name, const std::string& faustCode, int sr = 44100)
        : sampleRate(sr)
    {
        std::string errorMsg;
        factory = createDSPFactoryFromString(
            name, faustCode,
            0, nullptr,
            "",             // target: auto (LLVM)
            errorMsg,       // error output
            3               // opt level
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

        // Collect parameters
        FaustParamCollector collector;
        dspInstance->buildUserInterface(&collector);
        params = collector.params;
        paramLookup = collector.lookup;
    }

    virtual ~FaustBaseDSP() {
        if (dspInstance) delete dspInstance;
        if (factory) deleteDSPFactory(factory);
    }

    bool isValid() const { return dspInstance != nullptr; }

    int getNumInputs() const { return dspInstance ? dspInstance->getNumInputs() : 0; }
    int getNumOutputs() const { return dspInstance ? dspInstance->getNumOutputs() : 0; }
    int getSampleRate() const { return sampleRate; }

    // Parameter access
    bool setParam(const std::string& path, float value) {
        if (paramLookup.count(path)) {
            *paramLookup[path] = value;
            return true;
        }
        return false;
    }

    float getParam(const std::string& path) const {
        if (paramLookup.count(path)) {
            return *paramLookup.at(path);
        }
        return 0.f;
    }

    std::vector<FaustParam> getParams() const { return params; }
    dsp* getDSP() const { return dspInstance; }
};

// -------------------- Mono DSP --------------------
class FaustMonoDSP : public FaustBaseDSP {
public:
    FaustMonoDSP(const std::string& code, int sr = 44100)
        : FaustBaseDSP("MonoDSP", code, sr)
    {
        if (isValid() && (getNumInputs() != 1 || getNumOutputs() != 1)) {
            std::cerr << "[Faust] Warning: DSP is not mono (" 
                      << getNumInputs() << " in, " << getNumOutputs() << " out)\n";
        }
    }

    void process(float* input, float* output, int nframes) {
        if (!isValid()) return;
        FAUSTFLOAT* in[1]  = { input };
        FAUSTFLOAT* out[1] = { output };
        dspInstance->compute(nframes, in, out);
    }
};

// -------------------- Stereo DSP --------------------
class FaustStereoDSP : public FaustBaseDSP {
public:
    FaustStereoDSP(const std::string& code, int sr = 44100)
        : FaustBaseDSP("StereoDSP", code, sr)
    {
        if (isValid() && (getNumInputs() != 2 || getNumOutputs() != 2)) {
            std::cerr << "[Faust] Warning: DSP is not stereo ("
                      << getNumInputs() << " in, " << getNumOutputs() << " out)\n";
        }
    }

    void process(float* leftIn, float* rightIn,
                 float* leftOut, float* rightOut,
                 int nframes)
    {
        if (!isValid()) return;
        FAUSTFLOAT* in[2]  = { leftIn, rightIn };
        FAUSTFLOAT* out[2] = { leftOut, rightOut };
        dspInstance->compute(nframes, in, out);
    }
};
```

---

# 🧠 Usage Example (C++)

```cpp
#include "FaustDSP.hpp"
#include <vector>
#include <iostream>

int main() {
    std::string code = R"(
        import("stdfaust.lib");
        gain = hslider("Gain", 0.5, 0, 1, 0.01);
        freq = hslider("Freq", 440, 20, 20000, 1);
        process = os.osc(freq) * gain;
    )";

    FaustMonoDSP dsp(code, 48000);
    if (!dsp.isValid()) return 1;

    std::cout << "== Parameters ==\n";
    for (auto& p : dsp.getParams())
        std::cout << p.path << " = " << *p.zone
                  << " (" << p.min << "–" << p.max << ")\n";

    dsp.setParam("/Gain", 0.8f);
    dsp.setParam("/Freq", 1000.f);

    const int N = 64;
    std::vector<float> in(N, 0.0f), out(N);
    dsp.process(in.data(), out.data(), N);

    std::cout << "Processed block. Output[0] = " << out[0] << "\n";
}
```

---

# ✅ Summary of Capabilities

| Feature           | Description                                         |
| ----------------- | --------------------------------------------------- |
| **JIT Compile**   | Takes Faust source as string → compiles to LLVM DSP |
| **Mono/Stereo**   | Separate optimized wrappers                         |
| **Parameter API** | `getParam()`, `setParam()`, `getParams()`           |
| **Type Info**     | Each param includes label, path, min/max/init, type |
| **Pybind Ready**  | All members easily exposable to Python              |

---

If you’d like, I can now generate the **matching pybind11 module** so you can:

```python
dsp = pyfaust.FaustMonoDSP(code)
print(dsp.get_params())
dsp.set_param("/Gain", 0.9)
```

Would you like me to produce that next (with a full CMakeLists.txt so it builds cleanly)?
