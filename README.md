# AWS-Serverless-Exercise
Progetto AWS Serverless Exercise - Lambda, SQS and DynamoDB

Documento pdf con screenshot e passaggi scritti. 
[Progetto AWS Serverless Exercise.pdf](https://github.com/giadacontadino/AWS-Serverless-Exercise/files/13941074/Progetto.AWS.Serverless.Exercise.pdf)

(Gli screenshots sono presenti solo nel documento pdf)

1) Primo step: Creare la coda SQS 

Come primo passo, ho effettuato l'accesso al mio account personale e creato la coda SQS attraverso la quale passeranno i messaggi destinati alla tabella DynamoDB. 
Ho configurato una coda standard con le impostazioni predefinite, come illustrato negli screenshot seguenti.

2) Secondo step: Scrivere lo script di Python per il message producer

Come indicato nella email, ho riscontrato difficoltà nell'utilizzare Python sul mio computer a causa di problemi di path e Boto3. Per risolvere questo problema, ho optato per l'uso di AWS Cloud9, installando Boto3 tramite il terminale fornito dal servizio e caricando lo script nel file Python. 
Nello screenshot seguente è presente lo script utilizzato per inviare messaggi alla coda SQS precedentemente creata.
Innanzitutto, ho importato boto3 e ho creato il client SQS, seguendo le indicazioni trovate nella documentazione. 
Successivamente ho inserito il nome della coda SQS e ho fatto in modo che avviato lo script, l’URL della coda venisse automaticamente ottenuto. 
Infine ho inserito il testo del messaggio e attraverso la funzione print, ho fatto sì che ogni volta che lo script viene avviato, posso ottenere l’ID del messaggio per essere sicura che esso sia stato inviato alla coda.   

import boto3

queue_name = 'AWS-Serveless-Exercise'

sqs = boto3.client('sqs')

response = sqs.get_queue_url(QueueName=queue_name)
queue_url = response['QueueUrl']

response = sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='Hello4'
)

print(f"MessageId: {response['MessageId']}")

3) Terzo step: Creare la tabella di DynamoDB

Ho creato la tabella di DynamoDB per far sì che i messaggi inviati dallo script (message producer) vengono processati dalla funzione Lambda (consumer) e caricati automaticamente nella tabella.
Ho lasciato le configurazioni di default e ho indicato come chiave di partizione:  ‘message’, come negli screenshots che seguono. 

4) Quarto step: Creare IAM role per Lambda con le policy di autorizzazione

Prima di creare la funzione Lambda, ho creato il ruolo IAM da specificare durante il processo di creazione della funzione. 
Come negli screenshots che seguono, ho creato un ruolo IAM collegato al servizio Lambda. 
Ad esso ho collegato le policy di autorizzazione per SQS e DynamoDB e la policy AWS Lambda BasicExecutionRole per poter visualizzare i Cloudwatch metrics e i file log di Cloudwatch legati all’esecuzione della funzione. 

5) Quinto step: Creare la funzione di Lambda

Dopo la creazione del ruolo IAM, ho creato la funzione Lambda per registrare i messaggi che passano per la coda SQS nella tabella di DynamoDB. 
La funzione viene attivata ogni volta che viene avviato lo script, grazie al trigger che ho impostato dopo la creazione della funzione. 
Ho impostato python 3.12 per il runtime della funzione, dato che è il linguaggio di programmazione usato nel progetto e ho collegato il ruolo IAM creato nello step precedente. 

6) Sesto step: Scrivere la funzione Lambda (consumer) per conservare il messaggio su DynamoDB 

Dopo la creazione della funzione Lambda, ho caricato lo script python nella tab ‘code’ che dovrà registrare il messaggio inviato dal message producer (AWS Cloud9) nella tabella di DynamoDB. 
Innanzitutto ho importato json e boto3 e ho specificato la resource (DynamoDB) e le caratteristiche della tabella (nome e regione). 
Successivamente troviamo la funzione lambda_handler che prende il parametro event. Nella funzione, event contiene i dettagli dell'evento che ha innescato la funzione.
Dato che la funzione è chiamata dalla coda SQS, event contiene i record della coda. Successivamente, avviene l’estrazione dei record dall'evento. 
In questo caso, i record sono i messaggi nella coda. La variabile body viene utilizzata per estrarre il corpo del record.
Ho aggiunto il parsing del corpo del messaggio come JSON, a causa di errori trovati nei log di Cloudwatch, quando ho testato la funzione. Viene tentato il parsing del corpo come JSON utilizzando json.loads(body). 
Se il parsing ha successo, significa che il corpo è in formato JSON e viene sostituito alla variabile body. Se il parsing fallisce, il codice assume che il corpo sia una stringa normale e continua senza modifiche (non viene sostituito alla variabile body). 
La funzione ‘stampa’ (print) il corpo del messaggio ricevuto ed estrae i valori di messageId e message. 
Se il corpo è una stringa normale, il corpo del messaggio viene utilizzato come contenuto del messaggio, e messageId viene impostato su una stringa vuota.
Ho scelto di utilizzare questa distinzione cercata su internet per adattare la funzione a diverse strutture di messaggi provenienti dalla coda SQS, evitando alcuni errori che ho riscontrato durante i test. 
La funzione utilizza i valori estratti (params) per creare l’elemento da scrivere nella tabella DynamoDB. La funzione scrive quindi questo elemento nella tabella utilizzando table.put_item(). 
Se si verifica un'eccezione durante l'esecuzione di qualsiasi parte del codice, viene stampato un messaggio di errore e la funzione restituisce uno stato di errore (statusCode: 500) con un messaggio descrittivo, dato da +str(5). 
Se tutto il processo di scrittura su DynamoDB ha successo, la funzione restituisce uno stato di successo (statusCode: 200) con un messaggio che indica che la funzione è stata eseguita con successo.

import json
import boto3

dynamoDB = boto3.resource('dynamodb', region_name='eu-central-1')
table = dynamoDB.Table('Tabella-Serveless-Exercise')

def lambda_handler(event, context):
    try:
        print('event:', event)

        records = event['Records']

        body = records[0]['body']

        try:
            body = json.loads(body)
        except json.JSONDecodeError:
            pass

        print("Incoming message body from SQS :", body)

        if isinstance(body, dict):
            message_id = body.get('messageId', '')
            message_content = body.get('message', '')
        else:
            message_id = ''
            message_content = body

        params = {
            'TableName': 'Tabella-Serveless-Exercise',
            'Item': {
                'message': message_content,
            }
        }

        table.put_item(Item=params['Item'])

        print('Successfully written to DynamoDB')
        return {"statusCode": 200, "messageId": message_id, "message": "Successfully written to DynamoDB"}
 
    except Exception as e:
        print('Error in executing lambda:', str(e))
        return {"statusCode": 500, "message": "Error while execution: " + str(e)}

7) Settimo step: Attivare il trigger SQS per far sì che la funzione Lambda venga attivata ogni volta che lo script su AWS Cloud9 viene avviato

Per garantire che la funzione venga attivata ogni volta che SQS riceve un messaggio (in questo caso, il messaggio è inviato dallo script su AWS Cloud9), ho aggiunto il trigger di SQS alla funzione. 

8) Ottavo step: Collegare CloudWatch per effettuare il monitoring

Al fine di monitorare le prestazioni dei servizi utilizzati nel progetto (SQS, DynamoDB e Lambda), ho creato una dashboard su CloudWatch specificando i servizi da monitorare. 

9) Nono step: Verificare il corretto funzionamento della funzione attraverso i test e controllare i log di CloudWatch

Inizialmente, ho testato la funzione creando un evento di test con un messageId casuale e un body del messaggio contenente "Hello". 
Ho potuto constatare che la funzione ha lavorato correttamente e che il messaggio è stato salvato nella tabella DynamoDB. 
Successivamente, ho eseguito il test effettivo avviando lo script Python salvato su AWS Cloud9, modificando il body del messaggio in "Hello2" per verificare che il messaggio venisse salvato nella tabella.
Attraverso la tabella e i log di CloudWatch, ho verificato che la funzione è stata attivata all'invio del messaggio alla coda e che il messaggio è stato salvato nella tabella DynamoDB.

Optional requirements

1) Implement dead-letter queues for handling processing failures.

Per poter utilizzare una coda come dlq collegata alla main queue (in caso di errori durante il processo, i messaggi vengono indirizzati a questa coda e non alla principale), ho creato innanzitutto una nuova coda abilitata per essere utilizzata come dlq, come segue negli screenshots. 
Dopo la creazione della dead letter queue, ho selezionato su SQS la coda principale e ho cliccato edit nella tab ‘dead letter queue’. 
Ho abilitato l’opzione di mandare i 'undeliverable messages’ alla dead letter queue creata precedentemente.
Per testare la dlq, ho creato uno script che dà errore e che quindi manda il messaggio alla dlq. (Video test allegato nell'email)

import boto3
from botocore.exceptions import BotoCoreError, ClientError

sqs = boto3.client('sqs', region_name='eu-central-1')

main_queue_url = 'https://sqs.eu-central-1.amazonaws.com/339712995867/AWS-Serveless-Exercise'

dlq_url = 'https://sqs.eu-central-1.amazonaws.com/339712995867/dead_letter_queue'

def send_message_to_sqs(message_body):
    try:
        if "condizione_di_errore" in message_body:
            raise ValueError("Errore simulato")

        response = sqs.send_message(
            QueueUrl=main_queue_url,
            MessageBody=message_body
        )
        print("Messaggio inviato con successo:", response)

    except (BotoCoreError, ClientError, ValueError) as e:
        send_to_dlq(message_body)
        print("Errore durante l'invio del messaggio:", str(e))

def send_to_dlq(message_body):
    try:
        response = sqs.send_message(
            QueueUrl=dlq_url,
            MessageBody=message_body
        )
        print("Messaggio inviato alla DLQ:", response)

    except (BotoCoreError, ClientError) as e:
        print("Errore durante l'invio del messaggio alla DLQ:", str(e))

message_to_send = '{"message": "Hello", "condizione_di_errore": true}'
send_message_to_sqs(message_to_send)

2) Implement AWS X-Ray for tracing and analyzing the behavior of the Lambda, SQS and DynamoDB.

Innanzitutto modifico il ruolo IAM, aggiungendo la policy di autorizzazione per AWS X-Ray. 
Successivamente vado su Lambda, clicco nella tab Configuration della mia funzione, clicco a sinistra su monitoring and operations tools e infine clicco su edit per Additional monitoring tools. Abilito Active Tracing (AWS X-Ray). 
Infine controllo se nelle tracce di AWS X-Ray è stata registrata correttamente l’invocazione della funzione.
