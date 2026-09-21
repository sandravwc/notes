---
title: android/hexagon-npu-llm
---

- llm on hexagon v69 (8 gen 1 / 8+ gen 1) via executorch + qnn, from the adb shell

  ```sh
  # github.com/avisre/snapdragon-npu-llm, prebuilt bundle K9FxNa/Qwen3-0.6B-SM8450-Hybrid
  # their install.sh dies on the "+" in "8+ Gen 1" (eval) and on pip. by hand:
  # termux:
  cd ~/storage/shared/npu
  for f in hybrid_llama_qnn.pte qnn_llama_runner.zip tokenizer.json; do curl -sL -C - -O https://huggingface.co/K9FxNa/Qwen3-0.6B-SM8450-Hybrid/resolve/main/$f; done
  unzip -oq qnn_llama_runner.zip
  # workstation:
  D=/data/local/tmp/executorch_qualcomm_tutorial
  adb shell "mkdir -p $D && cp /sdcard/npu/qnn_llama_runner/* /sdcard/npu/tokenizer.json /sdcard/npu/hybrid_llama_qnn.pte $D/ && chmod 755 $D/*"
  adb shell "cd $D && export LD_LIBRARY_PATH=\$PWD ADSP_LIBRARY_PATH=\$PWD && ./qnn_llama_runner --decoder_model_version qwen3 --tokenizer_path tokenizer.json --model_path hybrid_llama_qnn.pte --prompt 'hi' --seq_len 160 --kv_updater SmartMask --eval_mode 1 --temperature 0.7; cat outputs.txt"
  # 8+ gen 1: prefill 242 tok/s, decode 28 tok/s, ttft 95 ms, rss 1 gb. cpu e4b for comparison: pp 24 / tg 4.9
  # tokenizer regex warning "(?!" from re2 is harmless
  ```

- limits, corrected

  ```txt
  8 mb vtcm is activation scratch, not a model size cap: v73 has the same 8 mb and runs 3b
  v69 has no int4 in the hmx unit. w4a16 still compiles for it (dequant on the fly), just not the fast path
  no vlm recipe for v69, vision starts at sm8550 in the executorch qualcomm examples
  bigger models = compile yourself. qualcomm account either way:
    ai hub: pip qai-hub, api token, `--device "Samsung Galaxy S22 5G" --precision w4a16` -> genie bundle   (saas)
    local:  qairt sdk + hexagon sdk (login to download only), executorch examples/qualcomm/oss_scripts/llama/llama.py -m SM8450   (offline after)
  decoders in the recipe: llama3.2 1b/3b, qwen2.5 1.5b, qwen3 0.6b, phi-3.5-mini 3.8b, gemma3 1b
  ```
