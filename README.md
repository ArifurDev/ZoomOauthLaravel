# Implementing Zoom in a Laravel Project

This guide provides a step-by-step process to integrate Zoom API functionality into a Laravel project using Server-to-Server OAuth for authentication and meeting creation. The implementation relies on the provided `ZoomMeetingTrait` and example usage code. We will not modify the existing code; instead, we'll explain how to incorporate it seamlessly.

## Prerequisites
- A Laravel project (version 8 or higher recommended).
- Composer installed.
- A Zoom account with a Server-to-Server OAuth app created (available in the Zoom App Marketplace).
- Basic knowledge of Laravel traits, controllers, and environment configuration.

## Step 1: Create a Zoom Server-to-Server OAuth App
1. Log in to the Zoom App Marketplace[](https://marketplace.zoom.us/).
2. Click "Develop" > "Build App" and select "Server-to-Server OAuth" app type.
3. Provide app details (name, description).
4. Note down the following credentials from the app's "App Credentials" section:
   - Account ID
   - Client ID
   - Client Secret
5. Activate the app and ensure scopes like `meeting:write:admin` are added for creating meetings.

## Step 2: Configure Environment Variables
Add the Zoom credentials to your Laravel project's `.env` file. Use the provided example format:

 - ZOOM_CLIENT_ID=your_client_id_here
 - ZOOM_CLIENT_SECRET=your_client_secret_here
 - ZOOM_ACCOUNT_ID=your_account_id_here
 - ZOOM_REDIRECT_URI=https://your-app-url.com (optional, but set if needed for other flows)



Ensure these are loaded using Laravel's `env()` helper.

## Step 3: Install Required Packages
The code uses GuzzleHttp for API requests and Firebase JWT (though not directly used in meeting creation, it's imported). Install them via Composer:

```code
composer require guzzlehttp/guzzle
```
```code 
composer require firebase/php-jwt
```


Also, ensure `illuminate/support` is available (standard in Laravel). For logging, Laravel's built-in Log facade is used.

## Step 4: Create the ZoomMeetingTrait
Create a new file in `app/Traits/ZoomMeetingTrait.php` and paste the provided trait code:

```php
<?php

namespace App\Traits;

use Firebase\JWT\JWT;
use GuzzleHttp\Client;
use Illuminate\Support\Facades\Log;

trait ZoomMeetingTrait
{
    use apiresponse;

    private $client;
    private $accessToken;

    public function __construct()
    {
        $this->client = new Client();
        $this->accessToken = $this->getAccessToken();
    }

    /**
     * Get Zoom API Access Token using Server-to-Server OAuth
     */
    private function getAccessToken()
    {
        $clientId     = env('ZOOM_CLIENT_ID');
        $clientSecret = env('ZOOM_CLIENT_SECRET');
        $accountId    = env('ZOOM_ACCOUNT_ID');

        if (empty($clientId) || empty($clientSecret) || empty($accountId)) {
            throw new \Exception('Zoom credentials are missing.');
        }

        try {
            $response = $this->client->post('https://zoom.us/oauth/token', [
                'form_params' => [
                    'grant_type' => 'account_credentials',
                    'account_id' => $accountId,
                ],
                'headers' => [
                    'Authorization' => 'Basic ' . base64_encode("$clientId:$clientSecret"),
                    'Content-Type' => 'application/x-www-form-urlencoded',
                ],
            ]);

            $data = json_decode($response->getBody()->getContents(), true);
            return $data['access_token'] ?? null;
        } catch (\Exception $e) {
            Log::error('Failed to retrieve Zoom access token: ' . $e->getMessage());
            return null;
        }
    }

    /**
     * Create a Zoom Meeting
     */
    public function createMeeting($data)
    {
        if (!$this->accessToken) {
            return $this->sendError('Failed to retrieve Zoom API token.',[],500);
        }

        try {
            $response = $this->client->post('https://api.zoom.us/v2/users/me/meetings', [
                'headers' => [
                    'Authorization' => 'Bearer ' . $this->accessToken,
                    'Content-Type' => 'application/json',
                ],
                'json' => [
                    'topic'       => $data['topic'],
                    'description' => $data['description'],
                    'type'        => 2,
                    'start_time'  => $this->toZoomTimeFormat($data['start_time']),
                    'duration'    => $data['duration'],
                    'timezone'    => 'UTC',
                    'settings'    => [
                        'host_video'        => filter_var($data['host_video'], FILTER_VALIDATE_BOOLEAN),
                        'participant_video' => filter_var($data['participant_video'], FILTER_VALIDATE_BOOLEAN),
                        'waiting_room'      => true,
                    ],
                ],
            ]);

            return json_decode($response->getBody(), true);
        } catch (\GuzzleHttp\Exception\RequestException $e) {
            return $this->sendError('Zoom API request failed.',[],500);

        }
    }

    /**
     * Convert time format to Zoom format
     */
    private function toZoomTimeFormat($dateTime)
    {
        try {
            $date = new \DateTime($dateTime);
            return $date->format('Y-m-d\TH:i:s\Z');
        } catch (\Exception $e) {
            Log::error('Error formatting Zoom time: ' . $e->getMessage());
            return '';
        }
    }
}
```

Note: The trait uses apiresponse trait (assumed to be custom for sending API responses). Ensure it's defined elsewhere in your project.

## Step 5: Use the Trait in a Controller
Incorporate the trait into a controller (e.g., MeetingController.php). Here's how to use it based on the provided example code. Assume you have a Meeting model and Carbon for date parsing (install nesbot/carbon if needed: composer require nesbot/carbon).
In your controller method (e.g., store for creating a meeting):

```php 

use App\Traits\ZoomMeetingTrait;
use Carbon\Carbon;
use App\Models\Meeting; // Assuming a Meeting model exists

class MeetingController extends Controller
{
    use ZoomMeetingTrait;

    public function store(Request $request)
    {
        // Validate input (assuming $validatedData comes from $request->validate())
        $validatedData = $request->validate([
            'title' => 'required|string',
            'description' => 'required|string',
            'date' => 'required|date',
            'time' => 'required|date_format:H:i',
            // Add other fields as needed
        ]);

        $meetingDate = Carbon::parse($validatedData['date']);
        // Check if the meeting date is today or in the future
        if (!$meetingDate->isToday() && !$meetingDate->isFuture()) {
            return $this->sendError('Meeting date must be today or in the future.');
        }

        $data = [
            'topic'      => $validatedData['title'],
            'description'=> $validatedData['description'],
            'start_time' => $validatedData['date'] . ' ' . $validatedData['time'],
            'duration'   => 60,
            'host_video' => 1,
            'participant_video' => 1,
        ];

        // Generate meeting schedule link
        $zoomMeeting = $this->createMeeting($data);

        if (empty($zoomMeeting) || !$zoomMeeting) {
            return $this->sendError('Zoom meeting not found.');
        }

        $zoomMeetingDate = Carbon::parse($zoomMeeting['start_time'])->format('Y-m-d');
        $zoomMeetingTime = Carbon::parse($zoomMeeting['start_time'])->format('h:i A');

        try {
            $meeting = Meeting::create([
                'meeting_id' => $zoomMeeting['id'],
                'user_id' => $order->user_id, // Adjust based on your context, e.g., auth()->id()
                'title' => $validatedData['title'],
                'description' => $validatedData['description'],
                'link' => $zoomMeeting['join_url'],
                'date' => $zoomMeetingDate,
                'time' => $zoomMeetingTime,
            ]);

            // Success response
            return $this->sendResponse([], 'Meeting created successfully.', 201);
        } catch (\Exception $exception) {
            Log::error('Zoom Meeting Creation Failed', [
                'error_message' => $exception->getMessage(),
                'data' => $data
            ]);
            return $this->sendError('Failed to create meeting', [], 422);
        }
    }
}
```

Adjust variables like $order->user_id to fit your application's context (e.g., use auth()->user()->id).
## Step 6: Create the Meeting Model and Migration
If not already present, create a Meeting model and migration:

```code 

php artisan make:model Meeting -m
```

In the migration file (e.g., create_meetings_table.php):

```php
Schema::create('meetings', function (Blueprint $table) {
    $table->id();
    $table->string('meeting_id');
    $table->unsignedBigInteger('user_id');
    $table->string('title');
    $table->text('description');
    $table->string('link');
    $table->date('date');
    $table->time('time');
    $table->timestamps();
});
```
Run php artisan migrate.

## Step 7: Handle API Responses
The code references $this->sendError and $this->sendResponse, which are likely from a custom apiresponse trait. If not defined, create it in app/Traits/apiresponse.php:

```php 

<?php

namespace App\Traits;

trait apiresponse
{
    public function sendResponse($data, $message, $code = 200)
    {
        return response()->json([
            'data' => $data,
            'message' => $message,
        ], $code);
    }

    public function sendError($message, $data = [], $code = 400)
    {
        return response()->json([
            'message' => $message,
            'data' => $data,
        ], $code);
    }
}
```
## Step 8: Testing the Implementation

 - Ensure your .env has valid Zoom credentials.
 - Use a tool like Postman to send a POST request to your meeting creation endpoint (e.g., /api/meetings) with JSON body:

```json
{
    "title": "Test Meeting",
    "description": "This is a test",
    "date": "2025-08-22",
    "time": "10:00"
}
```

 - Check the response for success and verify the meeting in your Zoom dashboard.
 - Monitor logs for errors (e.g., storage/logs/laravel.log).


## Troubleshooting

 - Token Retrieval Failure: Verify credentials and internet connectivity. Check Zoom API status.
 - Meeting Creation Errors: Ensure the user (me) has permission to create meetings.
 - Date/Time Issues: Use valid formats; the trait converts to Zoom's UTC format.
 - Rate Limits: Zoom has API rate limits; handle them in production.

This completes the integration. For advanced features like updating/deleting meetings, extend the trait similarly.
