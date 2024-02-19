# mlplatform_v0
There are configs and code for creating simple MLplatform

# Подготовка компонент
1) Создание виртуальной среды
~~~
mkdir project1 && cd project1/ && sudo apt install python3.10-venv
python3 -m venv mlplatform1 && source mlplatform1/bin/activate
mkdir jup && cd jup  #folder for jupyterhub files, start jupyter from this folder and run install command here
~~~
2) Установка JupyterHub
~~~
python3 -m pip install jupyterhub
sudo apt install npm
sudo npm install -g configurable-http-proxy
python3 -m pip install jupyterlab notebook # needed if running the notebook servers in the same environment
~~~
3) Запуск JupyterHub на порту 6000
~~~
jupyterhub --generate-config
~~~
изменяем конфигурцию:
~~~
jupyterhub_config.py
c.JupyterHub.port = 6000
c.JupyterHub.proxy_api_port = 6001
c.JupyterHub.ip = '0.0.0.0'
~~~
запускаем
~~~
sudo jupyterhub
~~~
доступ по ссылке http://<ip>:6000

4) Установка MLflow и Minio
~~~
pip install mlflow && wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20240204223613.0.0_amd64.deb -O minio.deb
sudo dpkg -i minio.deb
~~~
доступ по ссылке http://<ip>:5000 и http://<ip>:9000, креды в выводе запуска

5) Запуск Minio
~~~
minio server /home/neo/project1/minio --console-address :9001
~~~
В minio создать buсket: project1/ и Access Key: ???, Secret Key: ???

6) Настройка общения между Minio и Mlflow и запуск Mlflow
~~~
export AWS_ACCESS_KEY_ID=??? && export AWS_SECRET_ACCESS_KEY=??? && export MLFLOW_S3_ENDPOINT_URL=http://ip:9000
echo $AWS_ACCESS_KEY_ID && echo $AWS_SECRET_ACCESS_KEY && echo $MLFLOW_S3_ENDPOINT_URL
mlflow server --artifacts-destination s3://project1 --serve-artifacts --host 0.0.0.0
~~~
здесь,

--artifacts-destination s3://project1 - указываем какой бакет для артефактов в Minio
--serve-artifacts предоставляет пользовательский интерфейс для просмотра, загрузки и управления артефактами
ip - текущий ip хоста (158.160.119.67)

# Создание и регистрация модели
~~~
import mlflow
ip = "158.160.119.67"
# указывает URI по которому доступен MLflow
mlflow.set_tracking_uri(f"http://{ip}:5000")
 
#  создем класс модели на основе базового класса для моделей Python, совместимых с MLflow
class MyModel(mlflow.pyfunc.PythonModel):
# Метод predict в данном контексте является интерфейсом модели  
    def predict(self, context, model_input):
        return self.my_custom_function(model_input)
# Здесь описывается код, который выполняет модель, то есть алгоритм ее работы. Им и пользуется предикт.
    def my_custom_function(self, model_input):
        res = f"This is your input: {model_input}"
        return res
# Создаем экземпляр модели
my_model = MyModel()
 
# Путь по которому будет доступна модель и название директории.
model_path = "path_test_model1"
# Имя модели для регистрации
reg_model_name = "name_test_model1"
# начинает новый запуск MLflow с именем "test_model_run" и присваивает его переменной run.
with mlflow.start_run(run_name="test_model_run") as run:
    model_path = f"{model_path}-{run.info.run_uuid}"
# Сохраняетм модель my_model в указанный путь model_path с использованием MLflow.
    mlflow.pyfunc.save_model(path=model_path, python_model=my_model)
# Регистрируем модель в MLflow под указанным именем reg_model_name и сохраняем ее артефакты по указанному пути model_path.
    mlflow.pyfunc.log_model(artifact_path=model_path, python_model=my_model, registered_model_name=reg_model_name)
~~~
# Загрузка модели и отправка к ней запроса
Метод predict представляет собой часть модели MyModel, которая разрабатывается в среде MLflow.

При интеграции модели с MLflow, важно иметь метод predict. Он позволяет использовать модель в среде MlFlow для получения прогнозов на новых данных. Так MLflow может отслеживать метрики производительности модели, регистрировать модель и развертывать ее в проде.

После регистрации все модели доступны по пути. Тут /2 это вторая зарегестрированная версия модели. При каждой новой регистрации модели ее версия +1.
~~~
model_uri = f"models:/{reg_model_name}/2"
# Скачиваем модель
loaded_model = mlflow.pyfunc.load_model(model_uri)
# Передает ей через predict входные данные и сохраняем ее рез-тат в result
result = loaded_model.predict(100)
print(result)
~~~
#Ответ
Downloading artifacts: 100%
5/5 [00:00<00:00, 497.93it/s]
This is your input: 100

# Настройка сервера
Установка сервера
~~~
pip install mlserver mlserver-mlflow
~~~

Для корректно работы сервера необходимо сконфигурировать:

- настройки самого сервера в файле settings.json
- настройки модели которую он будет загружать в себя при запуске в файле model-settings.json

### settings.json
~~~
{
    "debug": "true",  // write output of starting process
    "http_port": "8000", // port for swagger, allowed by http://ip:8000/v2/docs    
    "grpc_port": "8001", // API порт
    "server_name":" mlserver" // name of server
}
~~~
### model-settings.json
~~~
{
    "name": "name_test_model1", //  model name (how you registered it in mlflow)
    "implementation": "mlserver_mlflow.MLflowRuntime", // tell mlserver type of model
    "parameters": {
        "uri": "s3://project1/0/84eb95eae9c4428283ef3afd5976c14b/artifacts/" // path to s3 where model saved
    }
}
~~~
Затем сервер запускается командой mlserver start .  из директории в которой лежат файлы настроек. 

# Модель:
Для того чтобы модель могла взаимодействовать с пользователем через mlserver и запросы, необходимо:

Написать программу, которая формирует запросы к модели и отправляет их ей через метдот POST по HTTP
Написать функцию, которая полученный от модели ответ будет приводить в понятный (визуально) формат
Внутри модели написать функции которые:
декодируют полученный от mlserver запрос в понятный модели формат данных
кодируют ответ модели в формат, который понимает mlserver
Формирование и отправка запросов модели через mlserver (программа)
Импорты
~~~
from requests import post, get
from requests.models import Response
from mlserver.codecs.string import StringRequestCodec
from mlserver.types import InferenceResponse, InferenceRequest
import json
~~~
Формирование тела запроса
~~~
def create_request_to_mlserver_model (data: dict, model_name: str) -> dict:
    j_input = json.dumps(data)
    dict_inference_request = {
        "inputs": [
            {
              "name": model_name,
              "shape": [len(j_input)],
              "datatype": "BYTES",
              "data": j_input,
            }
          ]
    }
    return dict_inference_request
 
d = {"a": 10}
ip = "158.160.52.53"
model_name = "work_model1"
host = f"http://{ip}:8000"
test_inf_resp = create_request_to_mlserver_model(d, model_name)
test_inf_resp
~~~
Функция create_request_to_mlserver_model получает словарь с данными и имя модели, затем создает тело запроса, который нужно передать в модель. Запрос представляет из себя python словарь, у которого в поле data строка json формата.

Отправка запроса
~~~
def send_request_to_mlserver_model(inference_request: dict, host: str, model_name: str) -> Response:
    endpoint = f"{host}/v2/models/{model_name}/infer"
    headers = {"Content-Type": "application/json"}
    post_response = post(endpoint, json=inference_request, headers=headers)
    return post_response #.text
 
send_request_to_mlserver_model(test_inf_resp, host, model_name)
~~~
Функция send_request_to_mlserver_model формирует запрос (обычный Response для POST запроса из модуля requests), в него передает словарь.

Однако predict получит Inference Request (так передача запросов происходит через  MLserver), поэтому нужно переформировать в доступный модели формат (уже внутри predict) 

Декодирование запроса (в коде модели) InferenceRequest → dict
Функция decode_request в классе модели должна форматировать полученный от mlserver тип данных InferenceRequest в dict понятный модели

Запрос который получает predict от MLserver
~~~
id='dd4e02f5-2b0c-4866-a5bf-125875c9ba84'
parameters=Parameters(content_type=None,
                      headers={
                          'host': '158.160.52.53:8000',
                          'user-agent': 'python-requests/2.31.0',
                          'accept-encoding': 'gzip, deflate, br',
                          'accept': '*/*',
                          'connection': 'keep-alive',
                          'content-type': 'application/json',
                          'content-length': '95',
                          'Ce-Specversion': '0.3',
                          'Ce-Source': 'io.seldon.serving.deployment.mlserver',
                          'Ce-Type': 'io.seldon.serving.inference.request',
                          'Ce-Modelid': 'work_model1',
                          'Ce-Inferenceservicename': 'mlserver',
                          'Ce-Endpoint': 'test_model7',
                          'Ce-Id': 'dd4e02f5-2b0c-4866-a5bf-125875c9ba84',
                          'Ce-Requestid': 'dd4e02f5-2b0c-4866-a5bf-125875c9ba84'})
inputs=[
    RequestInput(
        name='work_model1',
        shape=[9],
        datatype='BYTES',
        parameters=None,
        data=TensorData(__root__='{"a": 10}'))
]
outputs=None
 
его тип: <class 'mlserver.types.dataplane.InferenceRequest'>
~~~
В нем нас интересует блок inputs, где содержатся данные для обработки в модели. Его и вычленяем функцией decode_request.
~~~
decode_request
def decode_request(self, model_input: InferenceRequest) -> dict:
        raw_json = StringRequestCodec.decode_request(model_input) # Decode an inference request into a high-level Python object
        # raw_json = ['{"a": 10}'], тип raw_json: <class 'list'>
         _input = json.loads(raw_json[0])
        # raw_json[0] = {"a": 10}, тип raw_json[0] : <class 'str'>
        return _input
# Затем вызываем собственно функцию модели, которая обработает входные данные и вернет ответ
model_output = self.my_custom_function(model_input)
 
~~~
# Полностью predict выглядит вот так:
~~~
def predict(self, context, request_input):
        print(f"Полученный predict запрос: {request_input}, тип: {type(request_input)}")
        model_input = self.decode_request(request_input)
        print(f"Результат decode_request для передачи в модель: {model_input}, тип: {type(model_input)}")
 
        model_output = self.my_custom_function(model_input)
        print(f"Результат работы модели (my_custom_function): {model_output}, тип: {type(model_output)}")
 
        inference_response = self.encode_response_model(model_output)
        print(f"Обработанный ответ модели для передачи mlserver: {request_input}, тип: {type(request_input)}")
 
        return inference_response
~~~
# Энкодинг ответа модели
После того как модель вернет результат его нужно подготовить к отправке

Результат работы модели (my_custom_function): {'result': 20}, тип: <class 'dict'>
В mlserver передается словарь, который внутри сервера будет оборачиваться в InferenceResponse. Ждя этого необходимо передать ему объект в формате: {"output": array('{"result": 102}', dtype=object)}
~~~
encode_response_model
def encode_response_model(self, model_output: dict) -> dict:
    _output = json.dumps(model_output) #получаем JSON строку
    _output_np = np.array(_output, dtype='object') #кладем ее в array
    inference_response = {"output": _output_np} #формируем словарь-ответ
    return inference_response
~~~
Расшифровка ответа от POST запрос
После отправки запроса model_answer = send_request_to_mlserver_model(test_inf_resp, host, model_name) полученный ответ нужно преобразовать в понятный человеку вид - словарь.

Ответ
model_answer = send_request_to_mlserver_model(test_inf_resp, host, model_name)
model_answer :
[<Response [200]>]

Форматируем функцией decode_answer, которая из блока text в POST запросе вытаскивает содержимое поля data=TensorData(__root__=['{"result": 20}'])) и возвращает его в виде python dict.
~~~
decode_answer
def decode_answer(post_response: Response) -> dict:
    inf_resp = InferenceResponse.parse_raw(post_response.text)
    raw_json = StringRequestCodec.decode_response(inf_resp)
    res = json.loads(raw_json[0])
    return res
~~~
Промежуточные состояния данных такие:
~~~
inf_resp в decode_answer:
model_name='work_model1'
model_version=None
id='dd4e02f5-2b0c-4866-a5bf-125875c9ba84'
parameters=Parameters(
    content_type='dict',
    headers=None)
outputs=[
    ResponseOutput(
        name='output',
        shape=[1],
        datatype='BYTES',
        parameters=Parameters(
            content_type='np',
            headers=None),
        data=TensorData(__root__=['{"result": 20}']))
]
 
 
raw_json в decode_answer: ['{"result": 20}']
~~~
Результат:
~~~
output = decode_answer(model_answer)
output
{'result': 20}
~~~
