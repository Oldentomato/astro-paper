---
  author: jowoosung
  pubDatetime: 2025-11-27T04:54:12.028Z
  modDatetime: 2025-11-27T04:54:12.543Z
  title: vllm 서비스환경 구축
  slug: vllm-서비스환경-구축
  featured: false
  draft: false
  tags:
    - ai,infra
  description: vllm과 fastAPI를 이용한 LLM inference환경 구축
---
## Table of contents  

## 프로젝트  
해커톤 출품용 지역 큐레이팅용 LLM inference api 서버를 개발한 것이다.  
서울의 문화재나 자연관광 등의 데이터를 파인튜닝한 모델을 vllm과 fastAPI를 이용하여 서비스를 제공한다.  
LLM 베이스 모델은 unsloth기반 llama3.1 8b, 4bit 양자화한 모델이다.  
[레포지토리 이동하기](https://github.com/SCBBB-Hackathon/llm-service)  

## vllm  
vllm은 LLM을 고속으로 추론하고 높은 처리량과 낮은 지연 시간을 제공하기 위해 설계된 엔진이다.  
GPU메모리를 효율적으로 관리하는 PagedAttention 기술을 비롯하여, 스케줄링 최적화와 Speculative Decoding 등의 다양한 기능을 통해 LLM의 성능을 극대화하고 비용 효율성을 돕는 여러 옵션을 제공한다.  
### vllm으로 모델 호출  
vllm으로 베이스모델과 lora모델을 불러와 fastAPI app.state에 등록해야한다.  
우선 llm로드 클래스코드는 다음과 같다.  
```python
    def __init__(self):
        _culture_lora_path = getApiKey("CULTURE_LORA_PATH")
        _language_lora_path = getApiKey("LANG_LORA_PATH")
        self.lora_list = {
            "culture": LoRARequest("culture", 1, _culture_lora_path),
            "lang": LoRARequest("language", 2, _language_lora_path),
            "None": None
        }

        #tool 로드
        self.etiquette = Etiquette_Tool()
        self.websearch = WebSearchTool()

    def initialize_engine(
        self, model: str, quantization: str
    ) -> AsyncLLMEngine:
        """Initialize the LLMEngine."""

        tokenizer = AutoTokenizer.from_pretrained(getApiKey("LLM_BASE_MODEL_PATH"))
        tokenizer.pad_token = tokenizer.eos_token
        tokenizer.padding_side = 'right'

        structured_outputs_params = StructuredOutputsParams(choice=["지하철", "버스", "에스컬레이터", "식당"])

        eos_token_id = [tokenizer.eos_token_id,  tokenizer.convert_tokens_to_ids("<|eot_id|>")]
        self.sampling_params = SamplingParams(
            temperature=0.5, 
            top_p=0.9, 
            stop_token_ids=eos_token_id, 
            logprobs=1, 
            prompt_logprobs=1, 
            max_tokens=512, 
            repetition_penalty=1.2,   # 반복 억제 핵심
            output_kind=RequestOutputKind.DELTA
        )

        self.etiquette_sampling_params = SamplingParams(
            temperature=0.2, 
            top_p=0.9, 
            stop_token_ids=eos_token_id, 
            logprobs=1, 
            prompt_logprobs=1, 
            max_tokens=20, 
            repetition_penalty=1.2,   # 반복 억제 핵심
            output_kind=RequestOutputKind.DELTA,
            structured_outputs=structured_outputs_params
        )

        engine_args = AsyncEngineArgs(
            model=model,
            quantization=quantization,
            enable_lora=True,
            max_lora_rank=64,
            max_loras=4,
            max_model_len=2048,
            gpu_memory_utilization=0.8  # GPU 메모리 활용 비율 상향
        )
        
        self.tokenizer = tokenizer
        self.engine = AsyncLLMEngine.from_engine_args(engine_args)
```
서버배포를 위해 AsyncLLMEngine을 이용한다.  
그리고 각 요청마다 출력형태를 SamplingParams를 이용해 설정해준다.  
여기서 출력형태를 streaming하게 하고 싶다면 output_kind를 RequestOutputKind.DELTA로 설정해주면 된다.  
lora모델은 LoRARequest를 이용하여 딕셔너리형태로 미리 설정하고 generate할때 호출하는 형태로 구성했다.  
llm요청 코드는 다음과 같다.  
```python
    async def _run_llm(self, request_id, messages, lora_req):
        prompt = self.tokenizer.apply_chat_template(
            messages,
            tokenize=False,
            add_generation_prompt=True
        )

        try:
            async for output in self.engine.generate(
                request_id=request_id,
                prompt=prompt,
                sampling_params=self.sampling_params,
                lora_request=self.lora_list[lora_req]
            ):
                for completion in output.outputs:
                    new_text = completion.text
                    if new_text:
                        yield new_text

        except asyncio.CancelledError:
            await self.engine.abort(request_id)
            raise

        except Exception as e:
            raise RuntimeError(f"Error during LLM streaming: {e}") from e
```
해당 코드를 함수화 하여 각 요청에서 재사용할 수 있도록 구성한다.  
그리고 호출할때는 다음처럼 구성했다.  

```python
    async def process_requests_stream(self, request_id: str, query: str, lora_req: str):
        if self.websearch.check_place(query):
            messages = [
                {"role": "user", "content": f"{query}에 대해 알려주세요"}
            ]

            async for chunk in self._run_llm(request_id, messages, lora_req):
                yield chunk

        else:
            yield "잘 모르는 내용이라 웹에서 검색한 뒤 알려드리겠습니다...\n"
            search_result = await self.websearch.search_query(query)
            lora_req = "None"

            if search_result:
                messages = [
                    {"role": "user",
                    "content": f"{query}에 대해 검색한 결과는 {search_result}입니다. "
                                f"이 내용을 잘 정리해서 대화처럼 설명해주세요."}
                ]


                async for chunk in self._run_llm(request_id, messages, lora_req):
                    yield chunk
            else:
                yield "웹 검색에서 문제가 발생했습니다."


    async def process_requests(self, request_id: str, query: str, lora_req: str):
        chunks = [] 
        if self._check_place(query):
            messages = [
                {"role": "user", "content": f"{query}에 대해 알려주세요"}
            ]

            # JSON(dict) return
            async for chunk in self._run_llm(request_id, messages, lora_req):
                chunks.append(chunk)
            full_text = "".join(chunks).strip()
            return {"result": full_text}

        else:
            search_result = await self.websearch.search_query(query)
            lora_req = "None"

            if search_result:
                messages = [
                    {"role": "user",
                    "content": f"""{query}에 대해 검색한 결과는 {search_result}입니다. 
                                    이 내용을 잘 정리해서 대화처럼 설명해주세요."""}
                ]

                async for chunk in self._run_llm(request_id, messages, lora_req):
                    chunks.append(chunk)
                full_text = "".join(chunks).strip()
                return {"result": full_text}

            else:
                return {"error": "웹 검색에서 문제가 발생했습니다."}
```
stream과 json 응답형식 이렇게 2개로 나누었다.  

## 기능 워크플로우  
해당 지역에 관한 큐레이팅 요청 워크플로우이다.  
전체적으로 2가지로 나뉘어있는데, 하나는 지역명을 입력으로 큐레이팅을 요청하는 것이고,  
다른 하나는 해당 지역의 카테고리를 입력하면 그 지역에 대한 공공예절을 받아오는 것이다.  
### 큐레이팅 요청  
![이미지 설명](https://github.com/SCBBB-Hackathon/llm-service/blob/main/readme_imgs/qr_llm.png?raw=true)  
첫번째 큐레이팅 요청은 lora로 학습된 모델을 요청하는 것으로 학습된 범위 내에서 대답을 해준다.  
학습데이터에서 지역들을 리스트로 가져오고, 요청한 지역이 리스트에있으면 그냥 요청을, 없으면 web검색 mcp를 통해 정보를 가져오고 대답을 해주는 방식이다. 
### 대중교통 llm  
![이미지 설명](https://github.com/SCBBB-Hackathon/llm-service/blob/main/readme_imgs/trans_llm.png?raw=true)  
현재 위치와 목표위치를 입력으로 주면, 버스&지하철 가는방법과 비용 그리고 택시 이용법과 비용을 api를 통해 정보를 가져오고 답변해주는 방식이다. 

## 마무리  
현재 진행상황으로는 완전히 완성되어있지 않은 상태이다.  
챗봇을 위한 chat history저장, 대중교통 llm 대중교통 완성, 말투변경 lora 등 해야할 부분들이 많이 남아있다.  
추후에 완성하여 이 게시글을 다시 작성할 계획이다.






