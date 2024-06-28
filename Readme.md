# Core library for clean architecture in PHP

## Introduction

This documentation guides you through the utilization of the core library for implementing clean architecture in PHP.
We'll explore the creation of custom application request and use cases, paying special attention to handling missing and
unauthorized fields.

Practical examples are provided using code snippets from a test suite to showcase the library's usage in building a
modular and clean PHP application.

## Prerequisites

Ensure that you have the following:

- `PHP` installed on your machine (version `8.2.0 or higher`).
- `Composer` installed for dependency management.

## Installation

```
composer require ug-php/clean-architecture-core
```

## Core Overview

### Application Request

Requests serve as input object, encapsulating data from your http controller. In the core library, use the `\Urichy\Core\Request\Request` class
as the foundation for creating custom application request object. Define the expected fields using the `requestPossibleFields` property.

### Presenter

Presenters handle the output logic of your usecase. You have to extends `\Urichy\Core\Presenter\Presenter` and
implements `\Urichy\Core\Presenter\PresenterInterface`

### Usecase

Use cases encapsulate business logic and orchestrate the flow of data between requests, entities, and presenters.
Extends the `\Urichy\Core\Usecase\Usecase` class and implements `\Urichy\Core\Usecase\UsecaseInterface` with the execute method.

### Response

- Use `\Urichy\Core\Response\Response` to create usecase `response`.
- Supports success/failure status, custom message, HTTP status codes, and response data. 
- I recommend you to extends `\Urichy\Core\Response\Response` class to create your own response

## Example of how to use the core library

1. Creating a custom request and handling missing/unauthorized fields

- Extends `\Urichy\Core\Request\Request` and implements `\Urichy\Core\Request\RequestInterface` to create
  custom application request objects.
- Define the possible fields in the `requestPossibleFields` property.

```php
<?php

declare(strict_types=1);

use Urichy\Core\Request\Request;
use Urichy\Core\Request\RequestInterface;

interface CustomRequestInterface extends RequestInterface
{

}

final class CustomRequest extends Request implements CustomRequestInterface
{
    protected static array $requestPossibleFields = [
        'field_1' => true, // required parameter
        'field_2' => false, // optional parameter
    ];
}

> You can also apply constraints to the request fields. For that you have to
modify the `applyConstraintsOnRequestFields` methods as below:

final class CustomRequest extends Request implements CustomRequestInterface
{
    protected static array $requestPossibleFields = [
        'field_1' => true,
        'field_2' => true,
    ];
    
    /**
     * @param array<string, mixed> $requestData
     * @return void
     */
    protected static function applyConstraintsOnRequestFields(array $requestData): void
    {
        // if you use beberlei/assert library
        Assert::that($requestData['field_1'], '[field_1] field must not be an empty string.')->notEmpty()->string();
        Assert::that($requestData['field_2'], '[field_2] field must not be an empty string.')->notEmpty()->string();
        
        // example if you use symfony validator library
       
        /**
         * You have to import
         * use Symfony\Component\Validator\ConstraintViolationListInterface;
         * use Symfony\Component\Validator\Validation;
         */
        $validator = Validation::createValidator();
        $violations = $validator->validate($requestData, new Assert\Collection([
            'field_1' => [
                new Assert\NotBlank(message: '[field_1] can not be blank'),
                new Assert\Type(type: 'string', message: '[field_1] must be a string'),
                new Assert\Length([
                    'min' => 2,
                    'max' => 10,
                    'minMessage' => '[field_1] must be at least [{{ limit }}] characters long',
                    'maxMessage' => '[field_1] can not be longer than [{{ limit }}] characters',
                ])
            ],
            'field_2' => [
                new Assert\NotBlank(message: '[field_2] can not be blank'),
                new Assert\Type(type: 'string', message: '[field_2] must be a string'),
                new Assert\Length([
                    'min' => 2,
                    'max' => 10,
                    'minMessage' => '[field_2] must be at least [{{ limit }}] characters long',
                    'maxMessage' => '[field_2] can not be longer than [{{ limit }}] characters',
                ])
            ]
        ]));

        if (count($violations) > 0) {
            throw new BadRequestContentException(json_encode(self::buildError($violations)));
        }
    }
    
    /**
     * Builds an error array from the given ConstraintViolationListInterface if you use symfony validator library.
     * You can put this method into your abstract request. 
     *
     * @param ConstraintViolationListInterface $violations The list of constraint violations
     * @return array The error array built from the violations
     */
    private static function buildError(ConstraintViolationListInterface $violations): array
    {
        $errors = [];
        foreach ($violations as $violation) {
            $propertyPath = $violation->getPropertyPath();
            $errors[$propertyPath][] = $violation->getMessage();
        }

        return $errors;
    }
    
}

// when unauthorized fields
try {
    CustomRequest::createFromPayload([
        'field_1' => 1,
        'field_2' => 'value',
        'field_3' => new stdClass(),
    ]);
} catch (BadRequestContentException $exception) {
    // Handle unauthorized fields
    dd($exception->getErrors()); // ["field_3"]
}

// when missing fields
try {
    CustomRequest::createFromPayload([
        'field_2' => 'value',
    ]);
} catch (BadRequestContentException $exception) {
    // Handle missing fields
    dd($exception->getErrors()); // ["field_1" => "required"]
}

// when everything is good
$request = CustomRequest::createFromPayload([
    'field_1' => 1,
    'field_2' => 'value',
]);

dd($request->getRequestId()); // 6d326314-f527-483c-80df-7c157acdb95b
dd([
    'field_1' => $request->get('field_1'), 
    'field_2' => $request->get('field_2'),
    'unknown' => $request->get('unknown', 'default_value'),
]); // ['field_1' => 1, 'field_2' => 'value', 'unknown' => 'default_value']
dd($request->getRequestData()); // ['field_1' => 1, 'field_2' => 'value']

// or with nested request parameters
final class CustomRequest extends Request implements CustomRequestInterface
{
    protected static array $requestPossibleFields = [
        'field_1' => true, // required parameters
        'field_2' => true, // required parameters
        'field_3' => false, // optional parameters
        'field_4' => [
            'field_5' => [
                'field_6' => true, // required parameters
            ],
        ],
    ];
}

try {
    CustomRequest::createFromPayload([
        'field_1' => 1,
        'field_2' => 'value',
        'field_3' => new stdClass(),
        'field_4' => [
            'field_5' => [],
        ],
    ]);
} catch (BadRequestContentException $exception) {
    // Handle missing fields
    dd($exception->getErrors()); // ['field_4.field_5.field_6' => 'required']
}
```

2. Creating a custom usecase

- Extends `\Urichy\Core\Usecase\Usecase` and implements `\Urichy\Core\Usecase\UsecaseInterface` to create
  your use cases.
- Implement the `execute` method for the use case logic.

```php
<?php

declare(strict_types=1);

use Urichy\Core\Response\Response;
use Urichy\Core\Response\StatusCode;
use Urichy\Core\Usecase\Usecase;
use Urichy\Core\Usecase\UsecaseInterface;

interface CustomUsecaseInterface extends UsecaseInterface
{

}

// with request and presenter with empty response
final class CustomUsecase extends Usecase implements CustomUsecaseInterface
{
    public function execute(): void
    {
        // get request data field
        $requestData = $this->getRequestData(); // return your request data as array
        $requestData = [
            'field_1' => $this->getField('field_1'), 
            'field_2' => $this->getField('field_2'),
            'unknown' => $this->getField('unknown', 'default_value'),
        ];
        
        // process your business logic here
        
        // send usecase response without content
        // I recommend you to extends "Response" class to create your own response
        $this->presenter->present(Response::create());
    }
}

// or with response

final class CustomUsecase extends Usecase implements CustomUsecaseInterface
{
    public function execute(): void
    {
        $requestData = $this->getRequestData(); // return your request data as array
        $requestData = [
            'field_1' => $this->getField('field_1'), 
            'field_2' => $this->getField('field_2'),
            'unknown' => $this->getField('unknown', 'default_value'),
        ];
        
        // process your business logic here
        
        // send usecase response with content
        // I recommend you to extends "Response" class to create your own response
        $this->presenter->present(Response::create(
            success: true,
            statusCode: StatusCode::OK->value,
            message: 'success.response',
            data: [
                'field' => 'value',
            ]
        ));
    }
}

// without request
final class CustomUsecase extends Usecase implements CustomUsecaseInterface
{
    public function execute(): void
    {        
        // process your business logic here
        
        // send usecase response with content
        // I recommend you to extends "Response" class to create your own response
        $this->presenter->present(Response::create(
            success: true,
            statusCode: StatusCode::OK->value,
            message: 'success.response',
            data: [
                'field' => 'value',
            ]
        ));
    }
}

// with request and without presenter
final class CustomUsecase extends Usecase implements CustomUsecaseInterface
{
    public function execute(): void
    {
        // get request data
        $requestData = $this->getRequestData(); // return your request data as array
        
        // process your business logic here
        
        // $this->presenter will be null here
    }
}
```

3. Create the custom presenter

- Extends `\Urichy\Core\Presenter\Presenter` to create `presenters`.

```php
<?php

declare(strict_types=1);

use Urichy\Core\Presenter\Presenter;
use Urichy\Core\Presenter\PresenterInterface;

interface CustomPresenterInterface extends PresenterInterface
{

}

final class CustomPresenter extends Presenter implements CustomPresenterInterface
{
  // you can override parent methods here to customize them
}
```

4. Executing the Usecase

Instantiate your custom usecase, set your custom request and custom presenter, and execute the usecase.

```php
<?php

declare(strict_types=1);

use Urichy\Core\Exception\Exception;

try {
    $customRequest = CustomRequest::createFromPayload([
        'field_1' => 1,
        'field_2' => 'value',
    ]);
    $customPresenter = new CustomPresenter();
    $customUsecase = new CustomUsecase();
    $customUsecase
        ->setRequest($customRequest)
        ->setPresenter($customPresenter)
        ->execute();
    
    // now you can get usecase response from presenter
    $response = $customPresenter->getResponse();
    
    dd($response->isSuccess()); // true
    dd($response->getStatusCode()); // 200
    dd($response->getMessage()); // 'success.response'
    dd($response->getData()); // ['field' => 'value']
    dd($response->get('field')); // 'value'
    dd($response->get('unknown_field')); // null
    
    // or
    $response = $customPresenter->getFormattedResponse();
    dd($response);
    dd($response['status']); // success
    dd($response['code']); // 200
    dd($response['message']); // 'success.response'
    dd($response['data']); ['field' => 'value']
} catch (Exception $exception) {
    dd($exception);
}

// without request
try {
    $customPresenter = new CustomPresenter();
    $customUsecase = new CustomUsecase();
    $customUsecase
        ->setPresenter($customPresenter)
        ->execute();
    
    // now you can get usecase response from presenter
    $response = $customPresenter->getResponse();
    
    dd($response->isSuccess()); // true
    dd($response->getStatusCode()); // 200
    dd($response->getMessage()); // 'success.response'
    dd($response->getData()); // ['field' => 'value']
    dd($response->get('field')); // 'value'
    dd($response->get('unknown_field')); // null
    
    // or
    $response = $customPresenter->getFormattedResponse();
    dd($response);
    dd($response['status']); // error (if response status is false)
    dd($response['code']); // 400
    dd($response['message']); // 'error.response'
    dd($response['details']); ['field' => 'value']
} catch (Exception $exception) {
    dd($exception);
}

// without presenter and without request
try {
    $customUsecase = new CustomUsecase();
    $customUsecase->execute(); // return anything (void)
} catch (Exception $exception) {
    dd($exception);
}


// for exception, some method are available
dd($exception->getErrors()); // print details
[
  'details' => [
      'field_1' => 'required',
  ],
]
// or
[
  'details' => [
      'error' => 'field [username] is missing.',
  ],
]

dd($exception->getDetails()); // print error details
[
  'field_1' => 'required',
]

// or 

[
  'error' => 'field [username] is missing.',
],

dd($exception->getMessage()) // 'error.message'
dd($exception->getDetailsMessage()) // 'field [username] is missing.' only if error key is defined in details.

dd($exception->getErrorsForLog()) // print error with more context
[
  'message' => $this->getMessage(),
  'code' => $this->getCode(),
  'errors' => $this->errors,
  'file' => $this->getFile(),
  'line' => $this->getLine(),
  'previous' => $this->getPrevious(),
  'trace_as_array' => $this->getTrace(),
  'trace_as_string' => $this->getTraceAsString(),
]

dd($exception->format());
[
    'status' => 'success' or 'error',
    'error_code' => 400,
    'message' => 'throw.error',
    'details' => [
        'field_1' => 'required',
    ],
]
```

## Example with symfony controller

```php
<?php

declare(strict_types=1);

namespace Urichy\Controller;

use Urichy\Core\Exception\Exception;
use Urichy\Core\Response\StatusCode;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Request as SymfonyRequest;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/', name: "_app", methods: 'GET')]
final class CustomController extends AbstractController
{
    /**
     *  you don't have to have an interface to present it. You can directly
     *  inject the concrete class of your presenter into the controller's constructor
     */
    public function __construct(
        private readonly CustomPresenterInterface $customPresenter,
        private readonly CustomUsecaseInterface $customUsecase,
        private readonly CustomRequestInterface $customRequest
    ) {}

    public function __invoke(SymfonyRequest $request): JsonResponse
    {
        try {
            $this
                ->customUsecase
                ->setRequest($this->customRequest::createFromPayload([
                    'field_1' => $request->get('field_1'),
                    'field_2' => $request->get('field_2'),
                ])
                ->setPresenter($this->customPresenter)
                ->execute();
            
            $response = $this->customPresenter->getFormattedResponse();
        } catch (Exception $exception) {
            // you also can get errors for log: $exception->getErrorsForLog()
            return $this->json($exception->format(), $exception->getCode());
        }

        return $this->json($response, $response['code']);
    }
}

// or

#[Route('/', name: "_app", methods: 'POST')]
final class CustomController extends AbstractController
{
    public function __construct(
        private readonly CustomUsecaseInterface $customUsecase,
        private readonly CustomRequestInterface $customRequest
    ) {}

    public function __invoke(SymfonyRequest $request): JsonResponse
    {
        try {
            $this
                ->customUsecase
                ->setRequest($this->customRequest::createFromPayload([
                    'field_1' => $request->get('field_1'),
                    'field_2' => $request->get('field_2'),
                ])
                ->execute();
        } catch (Exception $exception) {
            // you also can get errors for log: $exception->getErrorsForLog()
            return $this->json($exception->format(), $exception->getCode());
        }

        return $this->json([], StatusCode::OK->getValue());
    }
}
```

## Example with laravel controller

```php
<?php

declare(strict_types=1);

namespace Urichy\Controller;

use Urichy\Core\Exception\Exception;
use Illuminate\Http\Request as LaravelRequest;
use Illuminate\Http\Response as LaravelResponse;

final class CustomController extends Controller
{
    public function __construct(
        private readonly CustomUsecaseInterface $customUsecase
    ) {}

    public function __invoke(LaravelRequest $request): LaravelResponse
    {
        try {
            $this->customUsecase->execute();
        } catch (Exception $exception) {
            // you also can get errors for log: $exception->getErrorsForLog()
            return response()->json($exception->format(), $exception->getCode());
        }

        return response()->json([], StatusCode::OK->getValue());
    }
}

// or

final class CustomController extends Controller
{
     /**
     *  you don't have to have an interface to present it. You can directly
     *  inject the concrete class of your presenter into the controller's constructor
     */
    public function __construct(
        private readonly CustomPresenterInterface $customPresenter,
        private readonly CustomUsecaseInterface $customUsecase
    ) {}

    public function __invoke(LaravelRequest $request): LaravelResponse
    {
        try {
            $this
                ->customUsecase
                ->setPresenter($this->customPresenter)
                ->execute();
            
            $response = $this->customPresenter->getResponse();
        } catch (Exception $exception) {
            // you also can get errors for log: $exception->getErrorsForLog()
            return response()->json($exception->format(), $exception->getCode());
        }

        return response()->json($response->getData(), $response->getStatusCode());
    }
}
```

## Units Tests

You also can execute unit tests.

```
$ make tests
```

## License

- Written and copyrighted ©2023-present by Ulrich Geraud AHOGLA. <iamcleancoder@gmail.com>
- Clean architecture core is open-sourced software licensed under the [MIT license](http://www.opensource.org/licenses/mit-license.php)