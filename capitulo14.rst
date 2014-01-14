=============================================
Capítulo 14: Sessões, Usuários, e Inscrição
=============================================

É hora de uma confissão: nós temos deliberadamente ignorado um aspecto importante de desenvolvimento web até esse ponto. Até agora, nós pensamos no tráfego visitando nossos sites como uma massa sem rosto e anônima se arremessando contra nossas páginas cuidadosamente projetadas.

Isso não é verdade, claro. Os browsers acertando nossos sites têm humanos reais por trás deles(na maioria das vezes, pelo menos). Isso é uma grande coisa a ignorar: a Internet está em seu melhor quando serve para conectar pessoas, não máquinas. Se nós vamos desenvolver sites realmente chamativos, eventualmente nós vamos ter que lidar com corpos atrás dos browsers.

Infelizmente, não é assim tão fácil. HTTP é projetado para ser *stateless* -- isto é, cada e toda requisição acontece em um vácuo. Não há persistência entre uma requisição e a próxima, e não podemos contar em nenhum aspecto da requisição(endereço IP, user agent, etc.) para consistentemente indicar requisições sucessivas da mesma pessoa.

Nesse capítulo você vai aprender a manusear essa falta de state. Nós vamos começar pelo level mais baixo(*cookies*), e trabalhar daí até ferramentas de nível mais alto para manusear sessões, usuários e inscrição.

Cookies
=======

Desenvolvedores de browser reconheceram que a falta de state do HTTP demonstra um grande problema para desenvolvedores Web há muito tempo, então *cookies* nasceram. Um cookie é um pequeno pedaço de informação que browsers guardam em favor dos servidores Web. Toda vez que o browser requisita uma página de um certo servidor, ele devolve um cookie que é inicialmente recebido.

Vamos dar uma olhada como isso pode funcionar. Quando você abre o browser and escreve "google.com", seu browser manda uma requisição HTTP para o Google que começa assim:
  
    GET / HTTP/1.1
    Host: google.com
    ...
    
Quando o Google responde, a resposta HTTP parece com assim:

    HTTP/1.1 200 OK
    Content-Type: text/html
    Set-Cookie: PREF=ID=5b14f22bdaf1e81c:TM=1167000671:LM=1167000671;
                expires=Sun, 17-Jan-2038 19:14:07 GMT;
                path=/; domain=.google.com
    Server: GWS/2.1
    ...
    
Note o header "Set-Cookie". Seu browser guarda aquele valor de cookie ("PREF=ID=5b14f22bdaf1e81c:TM=1167000671:LM=1167000671") e serve de volta para o Google toda vez que acessa o site. Então próxima vez que você acessa o Google, seu browser vai mandar uma requisição assim:

    GET / HTTP/1.1
    Host: google.com
    Cookie: PREF=ID=5b14f22bdaf1e81c:TM=1167000671:LM=1167000671
    ...
    
Google então usa esse valo do Cookie para saber que você é a mesma pessoa que acessou o site previamente. Esse valor pode, por exemplo, ser a chave para entrar no banco de dados que armazena essa informação. Google poderia(e faz) usar isso para mostrar seu nome de usuário na página.

Pegando e configurando Cookies
---------------------------

Quando lidando com persistência em Django, boa parte do tempo você vai querer usar a sessão de level alto e/ou frameworks de usuário discutidos posteriormente neste capítulo. No entanto, uma primeira olhada em como ler e escrever cookies em um level baixo. Isso deve ajudar você a entender como o resto das ferramentas discutidas neste capítulo realmente funciona, e vai ser útil se você já tiver precisado brincar com cookies diretamente.

Ler cookies que já estão configurados é simples. Todo objeto "HttpRequest" tem um um objeto"COOKIE" que funciona como um dicionário; você pode usá-lo para ler qualquer cookie que o browser mandou para a view::

    def show_color(request):
        if "favorite_color" in request.COOKIES:
            return HttpResponse("Your favorite color is %s" % \
                request.COOKIES["favorite_color"])
        else:
            return HttpResponse("You don't have a favorite color.")
            
Escrever cookies é um pouco mais complicado. Você precisa usar o método "set_cookie()" em um objeto "HttpResponse". Aqui está um exemplo do cookie "favorite_color" baseado em um parâmetro "GET"::

    def set_color(request):
        if "favorite_color" in request.GET:

            # Create an HttpResponse object...
            response = HttpResponse("Your favorite color is now %s" % \
                request.GET["favorite_color"])

            # ... and set a cookie on the response
            response.set_cookie("favorite_color",
                                request.GET["favorite_color"])

            return response

        else:
            return HttpResponse("You didn't give a favorite color.")

.. SL Tested ok

Você também pode passar um número opcional de argumentos para "response.set_cookie()" que controla aspectos do cookie, como mostrado na Tabela 14-1

.. table:: Tabela 14-1: Opções do Cookie

    ==============  ==========  =====================================================
    Parameter       Default     Description
    ==============  ==========  =====================================================
    ``max_age``     ``None``    Idade (em segundos) que um cookie deve durar.
                                Se este parâmetro for ``"None"`` o cookie vai durar
                                só até quando o browser for fechado.

    ``expires``     ``None``    O dia/tempo real que o cookie deve expirar.
                                Precisa estar no formato ``"Wdy, DD-Mth-YY 
                                HH:MM:SS GMT"``. se dado, este parâmetro sobrescreve,
                                o parâmetro ``"max_age"``.

    ``path``        ``"/"``     O prefixo do caminho para o qual este cookie é
                                válido. Browsers só vão passar os cookies de 
                                volta para as páginas abaixo desse prefixo de 
                                caminho, então você pode usar isso para prevenir
                                que cookies sejam mandados para outras sessões desse
                                site.

                                Isso é especialmente útil quando você não controla
                                o topo do domínio do seu site.

    ``domain``      ``None``    O domínio para o qual este cookie é válido. Você 
                                pode usar este parâmetro para configurar um cookie
                                cross-domain. Por exemplo, ``domínio=".example.com'``
                                can use this parameter to set a cross-domain
                                cookie. For example, ``domain=".example.com"``
                                vai configurar um cookie que é legível pelo domínio
                                ``www.example.com``, ``www2.example.com``, e
                                ``an.other.sub.domain.example.com``.

                                Se este parâmetro for ``None``, o cookie só vai ser
                                legível para o domínio que o criou.

    ``secure``      ``False``   Se for configurado para ``True``, este parâmetro
                                instrúi o browser a só retornar este cookie para 
                                páginas acessadas por HTTPS.
    ==============  ==========  =====================================================

As Benções Misturadas dos Cookies
-----------------------------

Talvez você perceber um número de problemas em potencial com a maneira que os cookies funcionam.
Vamos dar uma olhada em algum dos mais importantes:

* Armazenamento dos cookies é voluntário; um cliente não tem que aceitar ou 	armazenar cookies. Na verdade, todos os browsers permitem que os usuários controlem a política para aceitação de cookies. Se você quer ver o quão vitais os cookies são para a Web, tente ligar a opção "pergunte para aceitar cada cookie" do seu browser.

  Apesar de seu uso universal, cookies ainda são a definição de de não confiabilidade. Isso quer dizer que desenvolvedores devem checar que um usuário de fato aceita cookies antes de depender deles.

* Cookies(especialmente aqueles que não são mandados por HTTPS) não são seguros. Porque dados HTTP são mandados em texto claro, cookies são extremamente vulneráveis a ataques snooping. Isto é, um atacante snoopie conectado pode interceptar um cookie e lê-lo. Isso quer dizer que dados sensíveis nunca devem ser armazenados em um cookie.

 Existe um atáque ainda mais insíduo, conhecido como ataque *man-in-the-middle*, que é quando um atacante intercepta um cookie e o usa para parecer outro usuário. O capítulo 20 discute ataques dessa natureza com profundidade, assim como maneiras de prevení-los.

* Cookies não são seguros mesmo dos receptores esperados. A maioria dos browsers proveem maneiras fácies de editar o conteúdo de cookies individuais, e usuários com recursos sempre podem usar ferramentas como mechanize(http://wwwsearch.sourceforge.net/mechanize/) para manualmente construir requisições HTTP.

Então você não pode armazenar dados que possam ser sensíveis a modificaçOes em cookies. O erro canônico neste cenário é armazenar algo como "EstaLogado=1" em um cookie quando um usuário loga. Você ficaria impressionado com o número de sites que cometem erros dessa natureza; é necessário apenas um segundo para enganar o sistema "seguro" desses sites.

Framework de Sessão do Django
==========================

Com todas essa limitações e buracos de segurança em potencial, é óbvio que cookies e sessões persistentes  são exemplos desses "pontos de dor" no desenvolvimento Web. Mas é claro que o objetivo do Django é ser um anestésico efetivo, então ele vem com um framework de sessão feito para superar essas dificuldades.

Esse framework de sessão permite armazenar e carregar dados arbitrários a base de visitantes por site. Ele armazena dados do lado do servidor e abstrai o envio e recebimento de cookies. Cookies usam um ID de sessão hashed -- não os dados mesmo -- ou seja, protejendo você da maioria dos problemas de cookie.

Vamos olhar como permitir sessões e usá-las em views.

Permitindo Sessões
-----------------

Sessões são implementadas via um pedaço de middleware (ver capítulo 17) e um modelo Django. Para permitir sessões, você vai precisar seguir os seguintes passos: 

#. Edite as configurações das suss "MIDDLEWARE_CLASSES" e certifique-se que 
   "MIDDLEWARE_CLASSES" contém 
   "django.contrib.sessions.middleware.SessionMiddleware".

#. Certifique-se que ``'django.contrib.sessions'`` está na configuração dos seus ``INSTALLED_APPS`` (e rode ``manage.py syncdb`` se tiver que adicioná-lo).

A configuração esqueleto default criado por "startproject" tem ambos esses bits instalados, então a menos que você os tenha removido, você provavelmente não vai precisar mudar nada para permitir sessões.

Se você não quiser usar sessões, você pode querer remover a linha "SessionMiddleware" do "MIDDLEWARE_CLASSES" e "django.contrib.sessions" dos seus "INSTALLED_APPES". Só vai lhe salvar uma pequena quantidade de overhead, mas cada pequeno bit conta.

Usando Sessões em Views
-----------------------

Quando "SessionsMiddleware" é ativado, cada objeto "HttpRequest" -- o primeiro argumento de qualquer função view Django -- vai ter um atributo "sessão", que é um objeto parecido com um dicionário. Você pode ler e escrever nele da mesma maneira que um dicionário normal. Por exemplo, em uma view você pode fazer coisas como::

    # Configurar o valor da sessão:
    request.session["fav_color"] = "blue"

    # Pegar um valor de sessão -- isso pose set chamado em uma view diferente,
    # ou muitas requisições depois(ou ambos):
    fav_color = request.session["fav_color"]

    # Limpar um item da sessão:
    del request.session["fav_color"]

    # Checar se a sessão tem uma dada chave:
    if "fav_color" in request.session:
        ...

Você também pode usar outros métodos de dicionário como "keys()" e "items()" no "request.session".

Existem algumas regras simples para usar sessões Django efetivamente:

* Use strings Python normal como chaves de dicionários em "request.session" (em contraponto a integers, objects, etc.).

* Chaves de dicionários de sessão que começam com underscore são reservados para uso interno do Django. Na prática, o framework usa apenas poucas variáves de sessão prefixadas com underscore, mas a menos que você saiba o que eles são (e está disposto a se manter atuaizado com qualquer mudança no próprio Django), se manter longe do prefixo underscore vai manter Django de interferir com sua aplicação.

  Por exemplo, não use a chave de sessão chamada "fav_color", assim:
      request.session['_fav_color'] = 'blue' # Não faça isso!

*  Não substitua "request.session" com um novo objeto, e não acesse ou configure seues atributos. Use ele como um dicionário Python. Exemplos::

      request.session = some_other_object # Não faça isso!

      request.session.foo = 'bar' # Não faça isso!

Vamos dar uma olhada em alguns exemplos rápidos. Essa view simples configura uma  variável "has_commentend" como "True" depois do usuário postar um comentário. É uma maneira simples (se não particularmente segura) de previnir o usuário de postar mais de um comentário::

    def post_comment(request):
        if request.method != 'POST':
            raise Http404('Only POSTs are allowed')

        if 'comment' not in request.POST:
            raise Http404('Comment not submitted')

        if request.session.get('has_commented', False):
            return HttpResponse("You've already commented.")

        c = comments.Comment(comment=request.POST['comment'])
        c.save()
        request.session['has_commented'] = True
        return HttpResponse('Thanks for your comment!')

Essa view simples tem saída em um "member" do site::

    def login(request):
        if request.method != 'POST':
            raise Http404('Only POSTs are allowed')
        try:
            m = Member.objects.get(username=request.POST['username'])
            if m.password == request.POST['password']:
                request.session['member_id'] = m.id
                return HttpResponseRedirect('/you-are-logged-in/')
        except Member.DoesNotExist:
            return HttpResponse("Your username and password didn't match.")

E este aqui tem como saída um membro que logou via "login()" acima::

    def logout(request):
        try:
            del request.session['member_id']
        except KeyError:
            pass
        return HttpResponse("You're logged out.")

.. note::

	Na prática, essa é uma maneira feia de logar usuários. O framework 			autenticação discutido brevemente lida com essa tarefa para você de uma 		maneira muito mais robusta e útil. Esses exemplos são deliberadamente 		simples para que você possa facilmente ver o que está acontecendo.

Configurando Testes de Cookies
------------------------------

Como mencionado acima, você não pode depender de cada browser aceitar cookies. Então, como conveniência, Django provem uma maneira fácil de testar se o browser do usuário aceita cookies. Basta chamar "request.session.set_test_cookie()" em uma view, e cheque "request.session.test_cookie_worked()" em uma view subsequente -- não em uma mesma chamada de view.

Essa divisão estranha entre "set_test_cookie" e "test_cookie_worked()" é necessário graças a maneira que cookies funcionam. Quando um cookie é configurado, você não pode dizer se um browser o aceitou até a próxima requisição do browser.

É boa prática usar "delete_test_cookie()" para limpar depois de você mesmo. 
Faça isso depois de verificar que o cookie de teste funcionou.

Aqui está um exemplo de uso típico::

    def login(request):

        # If we submitted the form...
        if request.method == 'POST':

            # Check that the test cookie worked (we set it below):
            if request.session.test_cookie_worked():

                # The test cookie worked, so delete it.
                request.session.delete_test_cookie()

                # In practice, we'd need some logic to check username/password
                # here, but since this is an example...
                return HttpResponse("You're logged in.")

            # The test cookie failed, so display an error message. If this
            # were a real site, we'd want to display a friendlier message.
            else:
                return HttpResponse("Please enable cookies and try again.")

        # If we didn't post, send the test cookie along with the login form.
        request.session.set_test_cookie()
        return render(request, 'foo/login_form.html')

.. note::

	Novamente, as funções de autenticação prontas lidam com essa checagem para você.

Usando Sessões fora de Views
----------------------------

Internamente, cada sessão é apenas um modelo Django normal definido em "django.contrib.sessions.models". Cada sessão é identificado por um hash randômico de 32 caracteres armazenado em um cookie. Já que é um modelo normal, você pode acessar sessões usando a API de banco de dados normal do Django::

    >>> from django.contrib.sessions.models import Session
    >>> s = Session.objects.get(pk='2b1189a188b44ad18c35e113ac6ceead')
    >>> s.expire_date
    datetime.datetime(2005, 8, 20, 13, 35, 12)

Você precisa chamar "get_decoded()" para pegar os dados da sessão. Isto é necessário porque o dicionário é armazenado em um formado codificado::

    >>> s.session_data
    'KGRwMQpTJ19hdXRoX3VzZXJfaWQnCnAyCkkxCnMuMTExY2ZjODI2Yj...'
    >>> s.get_decoded()
    {'user_id': 42}

Quando Sessões são Salvos
-------------------------

Por default, Django só salva no banco de dados se a sessão tiver sido modificada -- isto é, se algum dos valores de dicionários tiverem sido atribuído ou deletado::


    # Session is modified.
    request.session['foo'] = 'bar'

    # Session is modified.
    del request.session['foo']

    # Session is modified.
    request.session['foo'] = {}

    # Gotcha: Session is NOT modified, because this alters
    # request.session['foo'] instead of request.session.
    request.session['foo']['bar'] = 'baz'

Para mudar esse comportamente default, configure "SESSION_SAVE_EVERY_REQUEST" para "True". Se "SESSION_SAVE_EVERY_REQUEST" for "True", Django vai salvar a sessão no banco de dados em cada requisição, mesmo que não sido modificado.

Note que o cookie da sessão só é mandado quando a sessão tiver sido criado ou modificada. Se "SESSION_SAVE_EVERY_REQUEST" for "True", o cookie da sessão vai ser mandado em cada requisição. Similarmente, a parte "expires" de um cookie de sessão é atualizado cada vez que o cookie de sessão é mandado.

Sessões de Tamanho de Browser(browser-length) vs. Sessões Persistentes
----------------------------------------------------------------------

Você deve ter percebido que o cookie mandado pelo Google para nós no começo desse capítulo continha "expires=Sun, 17-Jan-2038 19:14:07 GMT;". Cookies podem conter opcionalmente uma data de expiração que aconselha o browser em quando ele deve remover o cookie. Se um cookie não contem uma valor de expiração, o browser vai expirá-lo quando o usuário fechar a sua janela de browser. Você pode controlar o comportamento do framework de sessão nesse sentido com a configuração "SESSION_EXPIRE_AT_BROWSER_CLOSE".

Por default, "SESSION_EXPIRE_AT_BROWSER_CLOSE" é configurando para "False", o que quer dizer que cookies de sessão vão ser armazenados em browsers de usuários por "SESSION_COOKIE_AGE" segundos (que por default é duas semanas, ou 1,209,600 segundos). Use isso se você não quiser que pessoas tenham acesso cada vez que abrem o browser.

Se "SESSION_EXPIRE_AT_BROWSER_CLOSE" é configurado para "True", Django vai usar cookies de tamanho de browser(browser-length).

Outras Configurações de Sessão
------------------------------

Além de configurações já mencionadas, outras poucas configurações influenciam como o framework de sessão do Django usa cookies, como mostrado na Tabela 14-2.

.. table:: Table 14-2. Settings that influence cookie behavior

    ==========================  =============================  ==============
    Setting                     Description                    Default
    ==========================  =============================  ==============
    ``SESSION_COOKIE_DOMAIN``   O domínio para usar cookies    ``None``
                                de sessão. Configure isso para
 						  uma string como "example.com"
                                para cookies cross-domain,
                                ou use "None"para um cookie 
						  básico.

    ``SESSION_COOKIE_NAME``     O nome do cookie para user     ``"sessionid"``
						  sessões. Isto pose set qualquer
						  string. 

    ``SESSION_COOKIE_SECURE``   Usar ou não um cookie "seguro" ``False``
                                para o cookie de sessão. Se
                                esse cookie for configurado 
                                para ``True``, o cookie vai
                                set marcado como "seguro"                                						  isso significa que o browser                                				 		  vai guarantir que o cookie só 
                                será envied via HTTPS.
    ==========================  =============================  ==============

.. admonition:: Technical Details

	Para os curiosos, aqui estão algumas notas técnicas sobre os trabalhos internos do framework de sessão:

	* O dicionário de sessão aceita qualquer objeto Python ccapaz de ser 	             	  "picado"("pickled" em inglês). Veja a documentação feita pelo módulo 	     	  "pickle" feito no Python para informação sobre como isso funciona.

	* Os dados de sessão são armazenados em um banco de dados numa tabela com o 	  nome "django_session".

	* Dados de sessão são retornados de acordo com a demanda. Se você nunca 		  acessou "request.session", Django não vai acertar aquela tabela do banco 	  de dados.

	* Django só manda um cookie se precisar fazê-lo. Se você não configurar 		  qualquer dado de sessão, ele não vai mandar nenhum cookie de sessão (a 		  menos que "SESSION_SAVE_EVERY_REQUEST" esteja configurado para "True").

	* O framework de sessão é inteiramente, e apenas, baseado em cookie.
	  Ele não dá um passo para trás e colocar IDs e URLs da sessão como último 	  recurso, como outras ferramentas(PHP, JSP) fazem.

	  Essa é uma decisão de design intencional. Colocar sessões em URLs não só 	  faz a URL ficar feia, mas também faz o seu site ficar vulnerável a	 	 	  certos tipos de formulários de roubo de ID via o cabeçalho "Referer".

	Se vicê ainda está curioso, a fonte é bem direta; olhe em 	"django.contrib.sessions" para mais detalhes.

Usuários e Autenticação
=======================

Sessões nos dão uma maneira de persistir dados atráves de requisições múltiplas de browsers; a segunda parte da equação é usar essas sessões para login do usuário. Claro, não podemoes confiar que usuários são quem dizem ser, então precisamos autenticá-los no meio do caminho.

Naturalmente, Django prover ferramentas para lidar com essa tarefa comum (e muitas outras). O sistema de autenticação de usuário do Django lida com contas, grupos, permissões de usuários, assim como sessões de usuários baseadas em cookies. Esse sistema é comumemnte referido como um sistema *auth/auth* (autenticação e autorização). Esse nome reconhece que lidar com usuários é, comumente, um processo de dois passos. Nós precisamos

#. Verificar (*autenticar*) que um usuário é que ele ou ela diz ser
   (normalmente ao checar o nome de usuário e o password contra um banco de dados de usuários)
#. Verificar que o usuário é *autorizado* a fazer algumas operações
   (normalmente ao checar em uma tabela de permissões)

Continuando com essas necessidades, o sistema de auth/auth do Django consiste de um número de partes:

* *Users*: Pessoas registradas com seu site

* *Permissions*: Flags binárias(sim/não) informando se o usuário pode ou fazer certas tarefas

* *Groups*: Uma maneira genérica de aplicar rótulos e permissões para mais de um usuário

* *Messages*: Uma maneira simples de enfileirar e mostrar mensagens do sistema para usuários

Se você usou a ferramenta de administrado (discutida no Capítulo 6), você já viu muitas dessas ferramentas, e se você editou usuários ou grupos na ferramentas de administrador, você tem editado dados nas tabelas do banco de dados do sistema auth.

Permitindo Suporte de Autenticação
==================================

Como as ferramentas de sessão, suporte de autenticação é empacotado como uma aplicação Django em "django.contrib" que precisa ser instalado. Assim como as ferramentas de sessão, também é instalado por default, mas se você o removeu, você vai querer seguir os seguintes passos para instalar:

#. Verifique que o framework de sessão está instalado como descrito mais cedo nesse capítulo.
