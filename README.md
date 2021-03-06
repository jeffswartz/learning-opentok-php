# OpenTok Getting Started Sample App

A simple server that uses the OpenTok PHP SDK to create sessions, generate tokens for those
sessions, archive (or record) sessions and download those archives.

## Automatic deployment to Heroku

Heroku is a PaaS (Platform as a Service) that can be used to deploy simple and small applications
for free. To easily deploy this repository to Heroku, sign up for a Heroku account and click this
button:

[![Deploy](https://www.herokucdn.com/deploy/button.png)](https://heroku.com/deploy)

You can also install this repository on your own PHP server (see the next section).

## Installation

1. Clone this repository.

2. Use [Composer](https://getcomposer.org/) to install the dependencies:

    `$ composer install`

3. Next, input your own API Key and API Secret into the `run-demo` script file:

     export API_KEY=0000000
     export API_SECRET=abcdef1234567890abcdef01234567890abcdef

4. The run-demo file starts the PHP CLI development server (requires PHP >= 5.4) on port 8080. If you want to
run your server on another port, edit the file.Finally, start the server using the run-demo script:

     `$ ./run-demo`

5. Visit the URL http://localhost:8080/session in your browser. You should see a JSON response
containing the OpenTok API key,session ID, and token. Read through the sections below to understand
how the server has been implemented. Then proceed to install the [client application](https://github.com/opentok/opentok-js-getting-started)


Hint: If you get an error message that says "The sessionId must belong to the apiKey," you might
need to remove the `cache` directory in the sample app directory. This can happen if you try
running the app with a different API Key than you used the last time it ran.

# Walkthrough

This demo application uses the [Slim PHP micro-framework](http://www.slimframework.com/) and a
[light-weight caching library](https://github.com/Gregwar/Cache). These are similar to many other
popular web frameworks and data caching/storage software. These are not a requirement for using
OpenTok, but they simplify the code in this sample application.

## Main Controller (web/index.php)

The first thing done in this file is to require the autoloader, which pulls in all the dependencies that were installed by Composer. We now have the Slim framework, the Cache library, and most importantly the OpenTok SDK available.

    require $autoloader;
    
    use Slim\Slim;
    use Gregwar\Cache\Cache;
    

    use OpenTok\OpenTok;
    use OpenTok\Role;
    use OpenTok\MediaMode;

Next the controller performs some basic checks on the environment, initializes the Slim application ($app), and sets up the cache to be stored in the application's container ($app->cache).

The first thing that we do with OpenTok is to initialize an instance and store it in the application container. At the same time, we also store the OpenTok API key separately so that
the app can access it on its own.

Notice that the app gets the `API_KEY` and `API_SECRET` from the environment variables.


    // Initialize OpenTok instance, store it in the app contianer
    
    $app->container->singleton('opentok', function () {
        return new OpenTok( getenv('API_KEY'), getenv('API_SECRET'));
    });

    $app->apiKey = getenv('API_KEY');

The sample app uses a single session ID to demonstrate the video chat, archiving, and signaling functionality. It does not generate a new session ID for each call made to the server. Rather, it generates one session ID and stores it in the cache. In other applications, it would be common to save the session ID in a database table. If the cache does not have a session ID stored, like on the first run of the application, we use the stored OpenTok instance to create a Session. The call
to the `opentok->createSession()` method returns a Session object. The app calls `$session->getSessionId()` to return the session ID, it will be stored in the cache for later use.

    $sessionId = $app->cache->getOrCreate('sessionId', array(), function() use ($app ) {
        // If the sessionId hasn't been created, create it now and store it
        $session = $app->opentok->createSession(array(
                'mediaMode' => MediaMode::ROUTED
            ));

        return $session->getSessionId();
    });

**NOTE**: In order to clear the cache, just delete the cache folder created in your demo app directory.

Now we are ready to configure the routes for our sample app. We need four routes for our app - 

1. Generate a Session and Token
2. Start an archive
3. Stop an archive
4. View an archive

### 1. Generate a Session and Token

The route handler for generating a session and token is shown below. The session ID stored in the cache is used to generate a new token.

        // Route to return the SessionID and token as JSON
        
        $app->get( '/session', 'cors', function() use ( $app, $sessionId ) {

            $token = $app->opentok->generateToken( $sessionId );

            $responseData = array(
                'apiKey' => $app->apiKey,
                'sessionId' => $sessionId,
                'token'=>$token
            );

            $app->response->headers->set( 'Content-Type', 'application/json' );
            echo json_encode( $responseData );
        });

Next inside the route handler, we generate a token, so the client has permission to connect to that session. This is again done by accessing the stored OpenTok instance. Generate a new token for each client.

    $token = $app->opentok->generateToken( $sessionId );

Finally, we return the OpenTok API key, session ID, and token as a JSON-encoded string so that the client can connect to a OpenTok session.

### 2. Start an Archive

The handler for starting an archive is shown below. The session ID for which the archive is to be created is sent as a URL parameter by the client application. Inside the handler, the `startArchive()` method of the opentok instance is called with the session ID for the session that needs to be archived. The optional second argument is the archive name, which is stored with the archive and can be read later. 

    //Start Archiving and return the Archive ID
    
    $app->post('/start/:sessionId', 'cors', function ( $sessionId ) use ( $app ) {
    
        $archive = $app->opentok->startArchive( $sessionId, "Getting Started Sample Archive" );
        $app->response->headers->set( 'Content-Type', 'application/json' );

        $responseData = array( 'archive' => $archive );

        echo json_encode( $responseData );
    });

This causes the recording to begin. The response sent back to the client's XHR request will be the JSON-encoded string of the archive object.


### 3. Stop an Archive
    
Next we move on to the handler for stopping an archive:

    //Stop Archiving and return the Archive ID
    
    $app->post( '/stop/:archiveId', 'cors', function( $archiveId ) use ( $app ) {
            
        $archive = $app->opentok->stopArchive( $archiveId );
            
        $app->response->headers->set( 'Content-Type', 'application/json' );
        $responseData = array( 'archive' => $archive );
    
        echo json_encode( $responseData );
        
    });

This handler is similar to the handler for starting an archive. It takes the ID of the archive that needs to be stopped as a URL parameter (sent by the client application). Inside the handler, it makes a call to the `stopArchive()` method of the opentok instance which takes the archive ID as an argument. 

### 4. View an Archive

The code for the handler to view an archive is shown below.

    //Download the archive

    $app->get( '/view/:archiveId', 'cors', function( $archiveId ) use ( $app ) {
    
        $archive = $app->opentok->getArchive( $archiveId );

        if ( $archive->status=='available' )
            
            $app->redirect( $archive->url );
            
        else {
           
            $app->render( 'view.html', array(
                    'id' => $archive->id,
                    'status' => $archive->status,
                    'url' => $archive->url
           ));
        }
        
    });

Similar to the other archive handlers, this handler receives the ID of the archive to be downloaded (viewed) as a URL parameter from the client application. It makes a call to the `getArchive()` method of the opentok instance which takes the archive ID as the parameter. 

We then check if the archive is available for viewing. If it is available, the client application is redirected to the URL at which the archive is available for viewing.

    if ( $archive->status=='available' )
            $app->redirect( $archive->url );

If the archive is not yet available, we load a template file `templates/view.html` to which we pass the archive parameters (ID, status, and URL) . This template file checks if the archive is available. If not, it again makes a call to the /view handler.             

## Appendix -- Deploying to Heroku

Heroku is a PaaS (Platform as a Service) that can be used to deploy simple and small applications
for free. For that reason, you may choose to experiment with this code and deploy it using Heroku.

Use the button at the top of the README to deploy to Heroku in one click!

If you'd like to deploy manually, here is some additional information:

*  The provided `Procfile` describes a web process which can launch this application.

*  Provision the [RedisToGo addon](https://addons.heroku.com/redistogo). It is free for
   up to 5MB of data. Its configuration will be set for you automatically upon provisioning the
   service.

*  Use Heroku config to set the following keys:

   -  `OPENTOK_KEY` -- Your OpenTok API Key
   -  `OPENTOK_SECRET` -- Your OpenTok API Secret
   -  `SLIM_MODE` -- Set this to `production` when the environment variables should be used to
      configure the application. The Slim application will only start reading its Heroku config when
      its mode is set to `'production'`

   You should avoid committing configuration and secrets to your code, and instead use Heroku's
   config functionality.
