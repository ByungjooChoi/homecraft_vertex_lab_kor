# Homecraft Retail lab script:Elastic ESRE 와 Google GenAI를 이용한 전자상거래 검색 app 만들기

이 문서는 핸즈온 실습을 위한 스텝바이스텝 가이드이며, 다음 repo에서 작성된 검색 app 을 참조하고 있습니다. [Palm2 version](https://github.com/valerioarvizzigno/homecraft_vertex) / [Gemini version](https://github.com/valerioarvizzigno/homecraft_gemini). 

--- TESTED WITH ELASTIC CLOUD ELASTIC CLOUD 8.12 + ELAND 8.12 (and Elastic Cloud 8.8.1 + ELAND 8.3)   ---


## 사전 환경 설정

1. **Elastic Cloud Trial 신규 가입**
   
   Elastic Cloud 를 신규로 가입하여 [free trial account](https://www.elastic.co/cloud/elasticsearch-service/signup) 를 만듭니다
   신규 계정 생성시 gmail 계정을 사용하시고, _HongKildong@gmail.com_ 일 경우 "_HongKildong_**+codecamp**_@gmail.com_" 으로 만듭니다
   
3. **클러스터 설정**

   다음과 동일하게 Elastic 클러스터를 설정하세요:
   - GCP region: "🇰🇷 Seoul (asia-northeast3)", Elastic 버전 8.12, Autoscaling "None" 
   - 1-zone 8GB (memory) Hot node (or 2-zone x 4GB)
   - 1-zone 4GB Machine Learning node
   - 1-zone 1GB Kibana node
   - 1-zone 4GB Enterprise Search
   - Leave everything else as it is by default
   - Create the cluster and download/note down the admin username/password
   - Explore the Kibana console and the management console
  
   (If on the Elastic trial, you're not able to specify topology on the initial screen, so create the cluster as per suggested default and then go to "Edit My Deployment" in the left panel -> "Actions" menu -> "Edit deployment" and set it as the list before. Don't worry if some node size doesn't match exactly the previous, just max out everything)

 <img width="646" alt="image" src="https://github.com/ByungjooChoi/homecraft_vertex_lab_kor/assets/48340524/b0e05462-e6c7-445d-9b7f-c34a94ae4c15">



  ![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/7e11519d-1b73-4f19-93b2-bd7f166a72ca)



3. **머신러닝 노드 설정**

첫 번째 단계로, 색인할 콘텐츠에서 텍스트 임베딩을 생성하기 위해 Elastic ML 노드를 준비해야 합니다. 선택한 트랜스포머 모델을 Elastic에 로드하고 시작하기만 하면 됩니다. 이 작업은 [Eland Client](https://github.com/elastic/eland)를 통해 수행할 수 있습니다. 여기서는 [all-distillroberta-v1](https://huggingface.co/sentence-transformers/all-distilroberta-v1) ML 모델을 사용합니다. 이랜드 클라이언트를 실행하려면 도커가 설치되어 있어야 합니다. 파이썬/도커 설치 없이 이 단계를 쉽게 수행할 수 있는 방법은 구글의 클라우드 셸을 이용하는 것입니다. 복제하려는 eland 버전이 선택한 Elastic 버전과 호환되는지 확인하세요(예: 일반적으로 eland 8.12는 elastic Cloud 8.12와 정상 동작함)! 최신 Elastic 버전을 사용하는 경우, 일반적으로 복제하는 동안 Eland 릴리즈 버전을 지정할 필요가 없습니다.
   - On Kibana (left panel) --> Stack Management --> Security --> Users 메뉴에서 "elastic" 계정을 만들고 "superuser" role 을 붙이세요
     <img width="1102" alt="image" src="https://github.com/ByungjooChoi/homecraft_vertex_lab_kor/assets/48340524/94dd722e-5c7f-4f77-8487-046cba2927e6">

   - Google Cloud 콘솔로 들어가세요
   - Cloud Shell 에디터를 엽니다 (링크 클릭 [this link](https://console.cloud.google.com/cloudshelleditor?cloudshell=true))
   - 다음 명령을 입력하세요. 마지막 라인 user 생성시 password 를 임의로 변경한 경우 반드시 변경한 password 로 바꾸시고, 자신의 the elastic endpoint로 변경합니다(< > 표시도 삭제)
   - 패스워드에 ! 나 @ 등 특수문자가 있을 경우에 \ 로 escape 처리합니다

 ```bash
git clone https://github.com/elastic/eland.git #use -b vX.X.X option for specific eland version.

cd eland/

docker build -t elastic/eland .

docker run -it --rm elastic/eland eland_import_hub_model --url https://elastic:_codecamp_@<your_elastic_endpoint>:9243/ --hub-model-id sentence-transformers/all-distilroberta-v1 --start
 ```

   - Elatic admin home -> "Manage this deployment" 버튼을 클릭하여 Elasticsearch 의 "Copy endpoint" 를 클릭하면 자신의 endpoint 주소를 클립보드로 카피할 수 있습니다
   <img width="318" alt="image" src="https://github.com/ByungjooChoi/homecraft_vertex_lab_kor/assets/48340524/ff40e81a-1399-458d-9b5b-f5a087b8cf85">
   <img width="614" alt="image" src="https://github.com/ByungjooChoi/homecraft_vertex_lab_kor/assets/48340524/1bc3e1ee-04dd-43a6-85c6-7d76cb14c9a8">


  


4. **ML model 확인 및 테스트**

모델이 Elastic에 로드가 완료되면 배포를 입력하고 왼쪽 메뉴에서 "Stack Management" -> "Machine Learning"으로 이동합니다. all-distilroberta-v1 모델이 보이고 "started" 상태인 것을 확인하실 수 있습니다. "out of sync" 경고가 표시되면 해당 경고를 클릭하고 동기화합니다. 이제 모든 것이 설정되었습니다. ML 모델이 실행 중입니다. 이제 수집할 문서에 트랜스포머 모델을 적용할 수 있습니다. 원하는 경우 같은 페이지에서 모델 이름 오른쪽에 있는 점 3개 메뉴를 클릭하고 "Test model"을 선택하여 테스트할 수 있습니다.
   
   
5. **Ingest Part 1: The Web Crawler**
   
   Let's start with indexing some general retail data from the Ikea website. We will use the built-in web crawler feature, configure it this way:
   - Search for "Web Crawler" in the Elastic search bar
   - Name the new index as "search-homecraft-ikea" (watch out the "search-" prefix is already there) and go next screen.
   - Specify "https://www.ikea.com/gb/en/" in the domain URL field and click "Validate Domain". A warning should appear, saying that a robots.txt file was found. Click on it to open it in a separate browser tab and continue by clicking "Add domain"
   - For better crawling performance search the sitemap.xml filepath inside the robots.txt file of the target webserver, and add its path to the Site Maps tab.
   - To avoid crawling too many pages and stick with the english ones we can define some crawl rules. Set as follows (rule order counts!):
     
     ![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/1f0d52cc-2d01-4b5d-9c0e-927042ccd932)

6. Now the new empty index should be automatically created for you. Have a look at it:
   - From the left panel, in the "Enterprise Search" section, select "Content" and click on index to explore its details.
   - You can also query it from the "Management" -> "Dev Tools" UI with the following query
     ```bash
     #Query index config
     GET search-homecraft-ikea 
     #Query index content
      GET search-homecraft-ikea/_search

7. **Creating an ingest pipeline for inference**
    
   Before start crawling we need to attach an ingest pipeline to the newly create index, so every time a new document is indexed, it will be processed by our ML model that will enrich the document with an additional field, the vector representation of its content.
   - Open the index from the Enterprise Search UI and navigate to the "Pipelines" tab
   - Click on "Copy and customize" to create a custom pipeline associated to this index
   - In the Machine Learning section click on "Add Inference Pipeline"
   - Name the pipeline as "ml-inference-title-vector"
   - Select your transformer model and go next screen
   - Select the "Source field". This is the field that the ML model will process to create vectors from, and the UI suggests the ones automatically created from the web crawler. Select the "title" field as source field, specify "title-vector" as target field, leave everything else as default and go on until pipeline is created.
  
8. Check the newly created ingest pipeline searching for it from the "Stack Management" -> "Ingest pipelines" section. You are able to analyze the processors (the processing tasks) listed in the pipeline and add/remove/modify them. Note also that you can specify exception handlers.
 
   ![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/761cf843-c238-4f14-8fc2-47a6157f98b3)


8a. (Only if on Elastic 8.12, not the 8.3) As later we will be using "title-vector" field for kNN search in our application (and not the default "ml.inference.title-vector.predicted_values") let's add an additional SET processor at the bottom of the processors list as in the image below:

![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/871c5b93-1a8e-4337-b98c-447456351c89)


Be aware of adding the following condition to handle errors:

```bash
ctx?.ml?.inference != null && ctx.ml.inference['title-vector'] != null
```

For extra safety also add a REMOVE processor at the beginning of the processors list, to remove any occasional presence of "title-vector" field names from sources, as following:

![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/405bb7dd-80aa-48bf-aea8-ce8db5e0b607)


9. **Creating dense_vector mappings**
    
    Before launching the crawler we need to set the mappings for the target field where the vetors will be stored, specifying the "title-vector" field is of type "dense_vector", vector dimensions and its similarity algorithm. Let's execute the mapping API from the Dev Tools:

```bash
POST search-homecraft-ikea/_mapping
{
  "properties": {
    "title-vector": {
      "type": "dense_vector",
      "dims": 768,
      "index": true,
      "similarity": "dot_product"
    }
  }
}
```

10. **Launch the web crawler!**
    
    Start crawling: go back on the index and click on the "Start Crawling" button on the top right corner of the page. You will see the document count slowly increasing while the web crawler runs.

11. **Ingest Part 2: product catalog dataset upload**
    
    Let's now ingest a product catalog, to let our users search for products via hybrid search (keywords + semantics):
    - Load into Elastic this [Home Depot product catalog](https://www.kaggle.com/datasets/thedevastator/the-home-depot-products-dataset) via the "Upload File" feature (search for it in the top search bar). Click on "Import" and name the index "home-depot-product-catalog"
    - Take a look at the document structure, the fields available and their content. You will notice there are title and descriptions fields as well. We will apply the inference on the title field, so we can reuse the previously created ingest pipeline
    - From the Dev Tools console create a new empty index that will host the dense vectors called "home-depot-product-catalog-vector" and specify mappings.

```bash
PUT /home-depot-product-catalog-vector 

POST home-depot-product-catalog-vector/_mapping
{
  "properties": {
    "title-vector": {
      "type": "dense_vector",
      "dims": 768,
      "index": true,
      "similarity": "dot_product"
    }
  }
}
```

   - Re-index the product dataset through the same ingest pipeline previously created for the web-crawler. The new index will now have vectors embedded in documents in the title-vector field.

```bash
POST _reindex
{
  "source": {
    "index": "home-depot-product-catalog"
  },
  "dest": {
    "index": "home-depot-product-catalog-vector",
    "pipeline": "ml-inference-title-vector"
  }
}
```
  

  - (Note that we used these steps to show how to use reindexing and ingest pipelines via API. You can still apply the pipelines via UI as done before with the search-homecraft-ikea index)

12. **Ingest Part 3: data from BigQuery via Dataflow**
    
    To complete our data baseline we can also import an e-commerce order history. Leverage the BigQuery to Elasticsearch Dataflow's [native integration](https://www.elastic.co/blog/ingest-data-directly-from-google-bigquery-into-elastic-using-google-dataflow) to move this [sample e-commerce dataset](https://console.cloud.google.com/marketplace/product/bigquery-public-data/thelook-ecommerce?project=elastic-sa) into Elastic. Be aware the UI and available fields/label may visually change on Dataflow depending on the region you choose, but it should work regardlessly. This lab was tested with us-central1 region.
    - Click on "View Dataset" after reaching the Google Console with the previous link
    - Take a look at the tables available in this dataset within the BigQuery explorer UI. On the left panel open the "thelook_ecommerce" dataset and click on the "Order_items" table. Explore the content
    - Copy the ID (by clicking on the 3 dots) of the "Order_items" table and search for Dataflow in the Google Cloud service search bar
    - Create a new Dataflow job from template, selecting the "BigQuery to Elasticsearch" built-in template. You will need to specify:
       - The job's name
       - The region to run the job in (preferably choose the same you deployed the Elastic cluster in to minimize latencies and costs)
       - The Elastic cluster's "cloudID"
       - The Elastic API key to let Dataflow connect with Elastic(check [here](https://www.elastic.co/guide/en/kibana/current/api-keys.html) how to generate one)
       - The index name as "bigquery-thelook-order-items". Dataflow will automatically create this new index where all the table lines will be sent.
       - The dataset and table you want to read from (first field of the Optional Parameters): bigquery-public-data.thelook_ecommerce.order_items (some regions need bigquery-public-data:thelook_ecommerce.order_items instead - : after project name instead of .)
       - Elastic username and password
       - Launch the job
      
![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/8708b87f-5855-447d-b6de-db7632a515cb)


This will let users to submit queries for their past orders. We are not creating embeddings on this index because keyword search will suffice (e.g. searching for userID, productID). Wait for the job to complete and check the data is correctly indexed in Elastic from the indices list in the UI.

13. **Test Vector Search**
    
    Let's test a kNN query directly from the Kibana Dev Tools. We will later see this embedded in our app. You can copy paste the following query from [this file](https://github.com/valerioarvizzigno/homecraft_vertex/blob/main/Additional%20sources/helpful_elastic_api_calls).

![image](https://github.com/valerioarvizzigno/homecraft_vertex_lab/assets/122872322/1a6e0079-7b51-4791-adfe-6b863037a9b5)

We are executing the classic "_search" API call, to query documents in Elasticsearch. The Semantic Search component is provided by the "knn" clause, where we specify the field we want to search vectors into (title-vector), the number of relevant documents we want to extract and the user provided query. Note that, to compare vectors also the user query has to be translated into text-embeddings from our ML model. We are then specifying the "text_embedding" field in the API call: this will let create vectors on-the-fly on the user query and compare them with the stored documents.
The "query" clause, instead, will enable keyword search on the same elastic index. This way we are using both techniques but still receiving an unified output.
      
14. **Deploying the search app front-end on Compute Engine**
    
    We are now going to deploy our front-end app. Create a small Google Cloud Compute Engine linux machine with default settings, with public IPv4 address enabled (set it in the Advanced Settings -> Networking), HTTP and HTTPS traffic enabled (tick the checkboxes) and access it via SSH. This will be used as our web-server for the front-end application, hosting our "intelligent search bar". We suggest these settings:
    - e2-medium
    - Debian11  

15. Update the machine, install git, python and pip.
```bash
sudo apt-get update
sudo apt install git #Press Y if prompted
sudo apt install pip #Press Y if prompted

#Check if python is installed (it probably is).
python3 --version
#If not python version is found install it
sudo apt install python3

```

16. Clone the [homecraft_gemini source-code repo](https://github.com/valerioarvizzigno/homecraft_gemini) on your CE machine.

```bash
git clone https://github.com/valerioarvizzigno/homecraft_gemini.git
```

17. **Configure VertexAI and Elasticsearch SDKs**
    
     Install requirements needed to run the app from the requirements.txt file. After this step, close the SSH session and reopen it, to ensure all the $PATH variables are refreshed (otherwise you can get a "streamlit command not found" error). You can have a look at the requirements.txt file content via vim/nano or your favourite editor.

```bash
cd homecraft_vertex
pip install -r requirements.txt

```

18. Authenticate the machine to the VertexAI SDK (installed via the requirements.txt file) and follow the instructions.
```bash
    gcloud auth application-default login
```

19. Set up the environment variables cloud_id (the elastic CloudID - find it on the Elastic admin console), cloud_pass and cloud_user (Elastic deployments's user details) and gcp_project_id (the GCP project you're working in). This variables are used inside of the app code to reference the correct systems to communicate with (Elastic cluster and VertexAI API in your GCP project)

```bash
export cloud_id='<replaceHereYourElasticCloudID>'
export cloud_user='elastic'
export cloud_pass='<replaceHereYourElasticDeploymentPassword>'
export gcp_project_id='<replaceHereTheGCPProjectID>'
```

20. **Run the app!**
    
    Run the app and access it copying the public IP:PORT the console will display in a web browser page
```bash
streamlit run homecraft_home.py
```

21. Test your app!

## Web-app code 설명

genAI 모델이 어떻게 사용되는지, 그리고 VertexAI 및 Gemini Pro 모델과 통합하는 방법을 완전히 이해하려면 웹 앱 코드를 살펴보세요. 이 애플리케이션은 "streamlit" 파이썬 프레임워크로 구축되어 프로토타이핑과 데모를 위한 간단하고 빠른 프론트엔드 개발이 가능합니다

대부분의 코드는 애플리케이션을 시작하기 위해 "run" command에서 참조한 "homecraft_home.py" 파일에 있습니다. 추가 페이지는 "pages" 폴더에서 찾을 수 있으며, 이 폴더의 라우팅은 streamlit 에서 자동으로 관리합니다

소스 코드를 간략하게 설명하겠습니다:

- 첫 번째 줄은 필요한 라이브러리를 가져오는 것입니다. 앱 구조를 위한 "streamlit", API 호출과 Elastic 클러스터와의 연결을 추상화하기 위한 "elasticsearch", 그리고 모든 GenAI 모델 및 도구와 상호 작용하기 위한 Google의 SDK인 "vertexAI"를 가져오는 것입니다. 앞서 살펴본 바와 같이, 우리 앱이 VertexAI API를 호출할 수 있도록 하기 위해 Google 서비스에 대한 프로그래밍 방식의 액세스를 위해 머신을 인증해야 합니다.
- The first lines will be importing the requried libraries. "streamlit" for the app structure, "elasticsearch" for abstracting the API calls and connections with the Elastic cluster, and "vertexAI", the Google's SDK for interacting with all the GenAI models and tools. As previously seen, we had to authenticate our machine for programmatic access to Google services to let our app call VertexAI APIs.

```bash
import os
import pandas as pd
from io import StringIO
import streamlit as st
from elasticsearch import Elasticsearch
import vertexai
from vertexai.preview.generative_models import GenerativeModel, ChatSession, GenerationConfig, Image, Part
```

- 다음으로 Elastic과 GCP 프로젝트의 자격 증명/엔드포인트에 대해 이전에 정의한 환경 변수를 읽는 내부 변수를 설정합니다. 이러한 변수는 나중에 연결을 설정하는 데 사용됩니다.

```bash
projid = os.environ['gcp_project_id']
cid = os.environ['cloud_id']
cp = os.environ['cloud_pass']
cu = os.environ['cloud_user']
```

- 다음 설정으로 GenAI 모델을 구성합니다. 사용할 모델, 모델의 창의성 및 출력 길이를 정의하고 SDK를 초기화합니다.
- With the following settings we are configuring our GenAI model. We define which model we want to use, model's creativity and output lenght and we initialize the sdk.

```bash
generation_config = GenerationConfig(
    temperature=0.4, # 0 - 1. The higher the temp the more creative and less on point answers become
    max_output_tokens=2048, #modify this number (1 - 1024) for short/longer answers
    top_p=0.8,
    top_k=40,
    candidate_count=1,
)
vertexai.init(project=projid, location="us-central1")
model = GenerativeModel("gemini-pro")
visionModel = GenerativeModel("gemini-1.0-pro-vision-001")
```

- line 50 부터 line 185 까지는 Elastic 클러스터와의 연결을 설정하고 세 가지 주요 API 호출을 실행하는 메서드를 정의합니다. 
a. search_products: 시맨틱 검색을 사용하여 HomeDepot 데이터 세트에서 제품 찾기
b. search_docs: 시맨틱 검색을 사용하여 Ikea 웹 크롤링 인덱스에서 일반 소매 정보 찾기
c. search_orders: 키워드 검색으로 과거 사용자 주문 검색하기

- 진짜 마법은 다음 코드에서 일어납니다. 여기서는 웹 양식에서 사용자 입력을 캡처하고, Elastic에서 검색 쿼리를 실행하고, 응답을 사용하여 미리 정의된 프롬프트에 변수를 채워 GenAI 모델에 전송되도록 합니다.

```bash
# Generate and display response on form submission
negResponse = "I'm unable to answer the question based on the information I have from Homecraft dataset."
if submit_button:
    queryForElastic = ''
    answerVision = ''
    if uploaded_file is not None:
        visionQuery = 'What is in the picture?'
        answerVision = generateVisionResponse(visionQuery,uploaded_file)
        #To Elastic, for semantic search, we send both the question and the first answer from the vision model
        queryForElastic = query + answerVision 
        st.write(f"**Vision assistant answer:**  \n\n{answerVision.strip()}")
    
    es = es_connect(cid, cu, cp)
    resp_products, url_products = search_products(query if queryForElastic == '' else queryForElastic)
    resp_docs, url_docs = search_docs(query if queryForElastic == '' else queryForElastic)
    resp_order_items = search_orders(1) # 1 is the hardcoded userid, to simplify this scenario. You should take user_id by user session
    #prompt = f"You are an e-commerce AI assistant. Answer this question: {query} using this context: \n{resp_products} \n {resp_docs} \n {resp_order_items}."
    #prompt = f"You are an e-commerce AI assistant. Answer this question: {query}.\n If product information is requested use the information provided in this JSON: {resp_products} listing the identified products in bullet points with this format: Product name, product key features, price, web url. \n For other questions use the documentation provided in these docs: {resp_docs} and your own knowledge. \n If the question contains requests for user past orders consider the following order list: {resp_order_items}"
    prompt = [f"You are an e-commerce AI assistant.", f"You answer question around product catalog, general company information and user past orders", f"Answer this question: {query}.", f"Context: Picture content = {answerVision}; Product catalog = {resp_products}; General information = {resp_docs}; Past orders = {resp_order_items} "]
    #prompt = f"You are an e-commerce customer AI assistant. Answer this question: {query}.\n with your own knowledge and using the information provided in the context. Context: JSON product catalog: {resp_products} \n, these docs: \n {resp_docs} \n and this user order history: \n {resp_order_items}"
    #answer = vertexAI(chat, prompt)
    answer = generateResponse(prompt)

    if answer.strip() == '':
        st.write(f"**Elastic-powered Assistant:** \n\n{answer.strip()}")
    else:
        st.write(f"**Elastic-powered Assistant:** \n\n{answer.strip()}\n\n")
        #st.write(f"Order items: {resp_order_items}")
```

## Sample questions

### Following questions are for text only Gemini

하기 질문으로 테스트 해보세요: 

- "List the 3 top paint primers in the product catalog, specify also the sales price for each product and product key features. Then explain in bullet points how to use a paint primer".
You can also try asking for related urls and availability --> leveraging private product catalog + public knowledge

- "could you please list the available stores in UK" --> --> it will likely use (crawled docs)

- "Which are the ways to contact customer support in the UK? What is the webpage url for customer support?" --> it will likely use crawled docs

- Please provide the social media accounts info from the company --> it will likely use crawled docs

- Please provide the full address of the Manchester store in UK --> it will likely use crawled docs

- are you offering a free parcel delivery? --> it will likely use crawled docs

- Could you please list my past orders? Please specify price for each product --> it will search into BigQuery order dataset
  
- After uploading the image (find some in the samples/images folder) try queries like this one:

- List all the items I have bought in my order history in bullet points


### Following questions are for multimodal Gemini

After uploading the image (find some in the samples/images folder) try queries like this one:

- What's in the image? Do you have similar products in your catalog? If yes list them with descriptions and prices
